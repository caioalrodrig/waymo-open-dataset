# 15 — Direção end-to-end baseada em visão

| | |
|---|---|
| **Notebook** | [`tutorial_vision_based_e2e_driving.ipynb`](../../tutorial/tutorial_vision_based_e2e_driving.ipynb) |
| **Dataset** | End-to-End Driving (WOD-E2E), proto `E2EDFrame` em TFRecord |
| **Nível** | 5 · ★★★★☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md) (projeção em câmera), [04](04-deteccao-3d-camera-only.md), [10](10-predicao-de-movimento.md) |
| **Código-chave** | `protos/end_to_end_driving_data.proto`, `protos/end_to_end_driving_submission.proto`, `wdl_limited/camera/ops/py_camera_model_ops.py`, `metrics/python/rater_feedback_utils.py` |
| **Desafio original** | Vision-based End-to-End Driving (2025) |

---

## 1. O problema no mundo real

A pilha modular (percepção → predição → planejamento) tem interfaces escritas à
mão: caixas, classes, trajetórias. Tudo que não cabe nelas se perde — e erros de
um módulo se propagam para o próximo.

A abordagem **end-to-end** treina **um único modelo** que vai das câmeras direto
à trajetória que o carro deve seguir, aprendendo suas próprias representações
intermediárias. É a tendência atual, impulsionada por *foundation models* de
visão e linguagem.

O dataset WOD-E2E foca o ponto fraco de qualquer sistema: o **long tail** —
obras, desvios, objetos na pista, comportamentos incomuns de outros agentes —
situações raras em que não basta "imitar o comportamento médio".

---

## 2. Os dados

| Campo do `E2EDFrame` | Conteúdo |
|---|---|
| `frame.images` + `frame.context.camera_calibrations` | **8 câmeras** cobrindo 360° (FRONT, FRONT_LEFT, FRONT_RIGHT, SIDE_LEFT, SIDE_RIGHT, REAR_LEFT, REAR, REAR_RIGHT), com calibração |
| `past_states` | Histórico do ego: **4 s a 4 Hz** (posição, velocidade, aceleração) |
| `future_states` | Futuro do ego: **5 s a 4 Hz (20 waypoints)**, só posições — o alvo |
| `intent` | Intenção de alto nível do ego: `GO_STRAIGHT`, `GO_LEFT`, `GO_RIGHT` (ou `UNKNOWN`) — como um comando de navegação de GPS |
| `preference_trajectories` | (Validação) Trajetórias alternativas com **notas de avaliadores humanos de 0 a 10** |

Tudo já vem no **frame do veículo** (a pose é a identidade). Não há caixas 3D
nem mapa: o modelo precisa inferir tudo das imagens.

---

## 3. O que o notebook faz, passo a passo

1. Carrega o dataset e lê um `E2EDFrame`.
2. Decodifica e mostra as imagens; em especial as **3 câmeras frontais**.
3. Extrai o futuro do ego como matriz `(N, 3)` e **projeta os waypoints nas
   câmeras** com `py_camera_model_ops` — você vê o caminho que o motorista seguiu
   desenhado sobre a imagem.
4. **Submissão**: para cada frame de teste, uma **única trajetória** de 5 s a 4 Hz
   (`TrajectoryPrediction`) dentro de `FrameTrajectoryPredictions`, dividida em
   shards e empacotada em `.tar.gz`.
5. **Avaliação local**:
   - implementação simples de **ADE**;
   - cálculo do **Rater Feedback Score** com `rater_feedback_utils` no frame que
     contém as trajetórias avaliadas por humanos.

---

## 4. A técnica de IA

| Família | Ideia | Relação com o que você conhece |
|---|---|---|
| **Imitation learning direto** | CNN/ViT por câmera → fusão → MLP que regride os 20 waypoints, condicionado no `intent` e no `past_states` | Regressão supervisionada com backbone de visão |
| **BEV intermediário** | Levantar features de imagem para BEV (LSS/BEVFormer — ver [04](04-deteccao-3d-camera-only.md)) e planejar sobre ele | Detecção 3D + planejamento num só grafo diferenciável (ex.: UniAD) |
| **Modelos multimodais (VLM)** | Modelo de visão-linguagem que recebe imagens + comando e gera waypoints como texto/tokens | Fine-tuning de foundation models (ex.: EMMA, da Waymo) |
| **Multi-hipótese + seleção** | Gerar várias trajetórias candidatas e um *scorer* treinado para escolher | Combina com as notas humanas de preferência |

O `intent` resolve a ambiguidade de rota: sem ele, num cruzamento, "virar" e
"seguir reto" seriam igualmente corretos.

---

## 5. Como se mede sucesso: Rater Feedback Score (RFS)

Imitar o motorista não basta: em situações raras, **várias trajetórias podem ser
boas** e algumas próximas da gravada podem ser ruins. Por isso a métrica oficial
usa **preferências humanas**:

- Para cada cenário avaliado, avaliadores pontuaram algumas trajetórias
  candidatas (tipicamente 3) de 0 a 10.
- Em torno de cada trajetória avaliada há uma **região de confiança**: limiares
  lateral e longitudinal, verificados em 3 s e 5 s, que **crescem com a
  velocidade inicial** do ego (o longitudinal é 4× mais tolerante que o lateral).
- Se a trajetória prevista fica inteiramente dentro da região de uma trajetória
  avaliada, **herda a nota dela** (vale a melhor correspondência).
- Fora da região, a nota **decai exponencialmente** com a distância
  normalizada, com piso de 4.
- **RFS** = média dessas notas sobre os cenários (implementação em
  `metrics/python/rater_feedback_utils.py`).

O **ADE** (erro médio até o futuro gravado) é fornecido só como referência, para
autoavaliação.

> Essa é a mesma ideia de RLHF em LLMs: quando "a resposta certa" não é única,
> usa-se o julgamento humano como referência.

---

## 6. Armadilhas comuns

- **Uma única trajetória** por frame (não K como no Motion) — a submissão
  rejeita múltiplas.
- Frequência e horizonte fixos: 20 pontos a 4 Hz começando **após** o último
  frame fornecido.
- Só **um frame por segmento** (aprox.) tem `preference_trajectories`; o
  notebook mostra como encontrá-lo para calcular o RFS localmente.
- Imitar só o futuro gravado otimiza o ADE, não necessariamente o RFS.
- As operações de câmera usadas aqui estão em `wdl_limited/camera` (licença
  própria).

---

## 7. Fechando o roadmap

Este tutorial é uma síntese de tudo:

| Habilidade | Onde você aprendeu |
|---|---|
| Ler câmeras e calibrações, projetar 3D → imagem | [01](01-primeiro-contato-sensores.md), [04](04-deteccao-3d-camera-only.md) |
| Representações BEV | [03](03-mapas-hd-e-coordenadas.md), [11](11-occupancy-flow.md) |
| Prever trajetórias e lidar com multimodalidade | [10](10-predicao-de-movimento.md) |
| Avaliar comportamento por critérios humanos/segurança | [13](13-sim-agents.md), [14](14-scenario-gen.md) |

Próximos passos sugeridos:

- Treine um baseline: ResNet compartilhada nas 3 câmeras frontais → concatena com
  `past_states` e *one-hot* do `intent` → MLP para 20 waypoints. Compare ADE e
  RFS.
- Avalie o seu modelo em closed-loop usando as ideias do tutorial 13 (o que
  acontece quando o seu carro desvia do log?).

---

⬅️ [14 — Scenario Gen](14-scenario-gen.md) · 🏁 [Voltar ao roadmap](README.md)
