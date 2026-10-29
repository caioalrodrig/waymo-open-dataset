# 10 — Predição de trajetórias (Motion Prediction)

| | |
|---|---|
| **Notebook** | [`tutorial_motion.ipynb`](../../tutorial/tutorial_motion.ipynb) |
| **Dataset** | Motion (WOMD), formato `tf.Example` |
| **Nível** | 3 · ★★★☆☆ |
| **Pré-requisitos** | [09 — Ponte](09-ponte-perception-motion.md); [03](03-mapas-hd-e-coordenadas.md) ajuda |
| **Código-chave** | `protos/scenario.proto`, `protos/motion_metrics.proto`, `metrics/ops/py_metrics_ops.py` (`motion_metrics`), `data_conversion/` (Scenario → tf.Example) |
| **Desafio original** | Motion Prediction / Interaction Prediction |

---

## 1. O problema no mundo real

Para planejar uma conversão à esquerda, o AV precisa saber se o carro em sentido
contrário vai seguir reto, virar ou parar — e se o pedestre na esquina vai
atravessar. Planejar só com o "agora" é perigoso; planejar com uma previsão
única e errada também. O que se quer é:

- **Múltiplos futuros possíveis** por agente (o futuro é *multimodal*),
- cada um com uma **probabilidade**,
- para um horizonte de até **8 s**,
- **consistentes entre si** (dois carros previstos não deveriam colidir).

---

## 2. Os dados no formato `tf.Example`

O notebook usa a versão "tensorizada" de cada cenário, com tamanhos fixos:

| Grupo de features | Forma | Conteúdo |
|---|---|---|
| `roadgraph_samples/*` | `[30000, …]` | Mapa amostrado em pontos: `xyz`, `dir` (direção), `type`, `id`, `valid` |
| `state/past/*` | `[128, 10]` | 10 estados passados de até 128 agentes: `x, y, z, bbox_yaw, velocity_x/y, length, width, height, valid` |
| `state/current/*` | `[128, 1]` | Estado atual |
| `state/future/*` | `[128, 80]` | 80 estados futuros (o alvo) |
| `state/type`, `state/is_sdc`, `state/tracks_to_predict` | `[128]` | Tipo, se é o AV, se deve ser previsto |
| `traffic_light_state/*` | `[10/1/80, 16]` | Estados e posições de até 16 semáforos |

> O formato nativo é o proto `Scenario` (ver [09](09-ponte-perception-motion.md));
> o `tf.Example` é conveniente para TF, mas você pode gerar o seu próprio com
> `data_conversion/scenario_conversion`.

---

## 3. O que o notebook faz, passo a passo

1. Define o dicionário de features e cria um `tf.data.TFRecordDataset`.
2. **Visualiza** um cenário: plota o roadgraph e as posições dos agentes passo a
   passo (passado → atual → futuro) e monta uma **animação**. É o melhor jeito de
   ganhar intuição sobre o problema.
3. **Modelo de exemplo**: um MLP simples (`SimpleModel`) que recebe as 11
   posições `(x, y)` passadas/atual de cada agente e regride diretamente as 80
   posições futuras. O próprio notebook avisa: *"not at all competitive"*.
4. **Métricas**: embrulha `py_metrics_ops.motion_metrics` num
   `tf.keras.metrics.Metric` e configura `MotionMetricsConfig` (avaliação em
   3 s, 5 s e 8 s).
5. Treina por alguns passos e imprime as métricas.

---

## 4. A técnica de IA

O MLP ignora o mapa, os outros agentes e a multimodalidade. Os modelos reais
evoluíram assim:

| Geração | Representação da cena | Arquitetura | Exemplos |
|---|---|---|---|
| **Raster** | Imagem BEV com mapa e agentes pintados | CNN (como em classificação de imagens) | MultiPath |
| **Vetorial / grafo** | Polilinhas do mapa e trajetórias como conjuntos de vetores | GNN, atenção | VectorNet, LaneGCN |
| **Transformer de cena** | Tokens para agentes, faixas e semáforos | Transformer com atenção entre todos | Scene Transformer, Wayformer, MTR |
| **Geração autoregressiva** | Movimento discretizado em tokens | Modelo de linguagem sobre movimento | MotionLM |

**Multimodalidade** é tratada prevendo **K trajetórias** (K = 6 no desafio) com
um score cada, tipicamente com perda "*winner-takes-all*" (só a trajetória mais
próxima do real é penalizada na regressão) + classificação sobre qual foi a
vencedora, ou com âncoras/intenções pré-definidas (MultiPath).

> 💡 Para quem vem de detecção: é parecido com *anchor boxes* — cada "âncora" é
> um padrão de manobra (seguir reto, virar à esquerda...), e o modelo classifica
> e refina.

---

## 5. Como se mede sucesso

Avaliado nos agentes de `tracks_to_predict`, em 3 s, 5 s e 8 s, com até 6
trajetórias por agente:

| Métrica | Definição | Intuição |
|---|---|---|
| **minADE** | Menor erro médio de deslocamento (m) entre as K trajetórias e o real | "Pelo menos uma das suas hipóteses foi boa ao longo do caminho?" |
| **minFDE** | Idem, só no ponto final | "Acertou onde o agente termina?" |
| **Miss Rate** | Fração de agentes em que **nenhuma** hipótese ficou dentro de um limiar (lateral/longitudinal, que cresce com a velocidade) | Errar completamente uma manobra é o que mais importa para segurança |
| **Overlap Rate** | Fração de trajetórias de maior score que colidem com outros agentes | Previsões fisicamente impossíveis |
| **mAP / Soft mAP** | AP sobre as trajetórias ordenadas pelo score, por "bucket" de comportamento (reto, curva, U-turn, parado…) | **Métrica principal do ranking**: avalia se os scores são bem calibrados e se manobras raras são cobertas |

Por que *min* sobre K: o futuro real é só uma amostra de uma distribuição.
Penalizar uma hipótese plausível mas diferente do que aconteceu seria injusto;
o mAP complementa ao punir quem "chuta" muitas trajetórias com scores ruins.

---

## 6. Armadilhas comuns

- **Coordenadas globais**: normalize cada cenário (ou cada agente) para o frame
  do agente-alvo no instante atual (centrado, rotacionado pelo heading). Sem isso,
  o modelo precisa aprender invariâncias de translação/rotação sozinho.
- **Máscaras `valid`**: agentes aparecem e somem; posições inválidas valem
  `-1`. Ignore-as na perda e nas features.
- **`num_map_samples`**: se você gerar seu próprio `tf.Example`, ajuste o número
  de pontos do roadgraph (30.000 no padrão).
- Formatos de tensor para as métricas são específicos
  (`[batch, num_preds, top_k, agents_per_joint, steps, 2]`); o notebook mostra a
  forma exata.

---

## 7. Para ir além

- Adicione o mapa: selecione os N pontos de roadgraph mais próximos do agente e
  passe por um PointNet; concatene com o encoder de histórico.
- Troque a saída por K=6 trajetórias + scores com perda winner-takes-all e
  compare minADE/miss rate.
- Paper do dataset: *Large Scale Interactive Motion Forecasting for Autonomous
  Driving: The Waymo Open Motion Dataset* (Ettinger et al., ICCV 2021).

---

⬅️ [09 — Ponte](09-ponte-perception-motion.md) · ➡️ [11 — Occupancy & Flow](11-occupancy-flow.md)
