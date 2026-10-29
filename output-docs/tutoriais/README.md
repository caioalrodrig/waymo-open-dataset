# Roadmap dos Tutoriais — Waymo Open Dataset

> Para quem já sabe IA, CNNs e visão computacional, mas está chegando agora ao
> domínio de **veículos autônomos (AV)**.

Este diretório transforma os 17 notebooks de [`tutorial/`](../../tutorial/) em uma
**trilha de estudo em ordem didática**. Cada guia explica:

1. **O problema no mundo real** que aquele recorte de dados ajuda a resolver;
2. **Os dados**: que "fatia" do dataset é usada e qual o seu formato;
3. **O que o notebook faz**, passo a passo;
4. **A técnica de IA** por trás, ligada ao que você já conhece de CV/DL;
5. **Como se mede sucesso** (métricas) e por que elas são desenhadas assim;
6. Armadilhas, exercícios e o próximo passo.

---

## 1. Que tipo de dúvida este repositório resolve?

O repositório **não é um modelo de direção autônoma**. Ele é a "infraestrutura
de pesquisa" em torno de três datasets: formatos de dados (protobuf, TFRecord,
Parquet), utilitários de leitura e geometria, e — principalmente — o **código
oficial de avaliação** dos desafios da Waymo. Na prática, ele responde perguntas
dos seguintes tipos:

| Tipo de pergunta | Exemplos concretos | Onde no roadmap |
|---|---|---|
| **"Como os dados estão organizados?"** | O que há num `Frame`? O que é uma *range image*? Como juntar caixas 3D com imagens de câmera no formato v2? | [01](01-primeiro-contato-sensores.md), [02](02-formato-v2-parquet.md) |
| **"Como converto entre sistemas de coordenadas?"** | Como projetar um ponto LiDAR numa imagem? Como alinhar nuvens de pontos de frames diferentes num mapa? | [01](01-primeiro-contato-sensores.md), [03](03-mapas-hd-e-coordenadas.md), [04](04-deteccao-3d-camera-only.md) |
| **"O que está ao redor do carro agora?"** (percepção) | Onde estão os carros/pedestres em 3D? Que ponto pertence à rua, à calçada, à vegetação? Qual a postura deste pedestre? | [04](04-deteccao-3d-camera-only.md)–[08](08-object-assets-reconstrucao.md) |
| **"O que vai acontecer nos próximos segundos?"** (predição) | Para onde este ciclista vai em 8 s? Que regiões da rua estarão ocupadas, inclusive por quem eu ainda não vejo? | [10](10-predicao-de-movimento.md)–[12](12-womd-multimodal-lidar-camera.md) |
| **"Como testar um carro autônomo sem ir para a rua?"** (simulação) | Como simular outros motoristas de forma realista e reativa? Como gerar cenários de tráfego plausíveis do zero? | [13](13-sim-agents.md), [14](14-scenario-gen.md) |
| **"E se um único modelo dirigir a partir das câmeras?"** (end-to-end) | Dadas as câmeras e uma intenção ("virar à esquerda"), qual trajetória o carro deve seguir? | [15](15-e2e-driving.md) |
| **"Meu modelo é bom? Como comparo com o estado da arte?"** | Como calcular mAP/APH, LET-AP, mIoU, wSTQ, OKS, minADE, Soft-IoU, *metametric*, RFS? Como montar a submissão para o leaderboard? | Todos — cada guia tem uma seção de métricas |

---

## 2. Como o Motion Dataset complementa o dataset de Perception?

> Interpretamos "training dataset" como o **Perception Dataset** — o conjunto de
> sensores brutos + rótulos humanos usado para *treinar modelos de percepção*.
> (Atenção: `training` também é o nome de um *split* em todos os datasets.)

Resposta curta:

- O **Perception** responde *"o que existe ao meu redor neste instante?"*. Ele
  entrega **pixels e pontos 3D** com rótulos humanos densos, mas tem apenas
  ~1.150 segmentos de 20 s — pouco para aprender comportamento.
- O **Motion (WOMD)** responde *"como esses agentes vão se mover?"*. Ele
  **abstrai os sensores** e entrega o mundo já "percebido": caixas com
  trajetórias, mapa vetorial e semáforos, em **103.354 segmentos** (centenas de
  horas), garimpados para conter interações interessantes.
- **A saída da percepção é a entrada da predição.** Os dois datasets
  compartilham convenções (10 Hz, tipos de objeto, `map.proto`, frames de
  coordenadas), então formam dois andares consecutivos da mesma pilha.
- Desde a v1.2, o Motion também traz **LiDAR e tokens de câmera** do 1º segundo
  de cada cenário — uma ponte para modelos que vão do sensor à previsão.

A explicação completa, com diagramas e tabela comparativa, está em
**[09 — Ponte Perception ↔ Motion](09-ponte-perception-motion.md)**.

---

## 3. O roadmap

Comece pelo **[00 — Panorama](00-panorama-veiculos-autonomos.md)**: ele explica a
pilha de software de um AV e situa cada dataset nela. Depois siga os níveis.

```
NÍVEL 0 ─ Panorama do domínio
   00  Panorama: a pilha de um veículo autônomo
        │
NÍVEL 1 ─ Fundamentos de dados e geometria (Perception)
   01  Primeiro contato: câmeras, LiDAR, range images ....... tutorial.ipynb / tutorial_local.ipynb
   02  Formato v2 (Parquet) e engenharia de dados ........... tutorial_v2.ipynb
   03  Mapas HD e sistemas de coordenadas ................... tutorial_maps.ipynb
        │
NÍVEL 2 ─ Tarefas de percepção (o que existe ao redor?)
   04  Detecção 3D só com câmera (camera-synced boxes, LET) . tutorial_camera_only.ipynb
   05  Segmentação semântica 3D de nuvens de pontos ......... tutorial_3d_semseg.ipynb
   06  Segmentação panóptica panorâmica em vídeo ............ tutorial_2d_pvps.ipynb
   07  Estimativa de pose humana 2D/3D ...................... tutorial_keypoints.ipynb
   08  Object assets: reconstrução 3D / NeRF por objeto ..... tutorial_object_asset.ipynb
        │
   09  ★ PONTE: do Perception ao Motion Dataset (leitura conceitual)
        │
NÍVEL 3 ─ Predição de comportamento (o que vai acontecer?)
   10  Predição de trajetórias multimodal ................... tutorial_motion.ipynb
   11  Occupancy & Flow: predição densa em grade BEV ........ tutorial_occupancy_flow.ipynb
   12  WOMD multimodal: LiDAR e tokens de câmera ............ tutorial_womd_lidar.ipynb / tutorial_womd_camera.ipynb
        │
NÍVEL 4 ─ Simulação em malha fechada (como testar?)
   13  Sim Agents: simular o tráfego de forma realista ...... tutorial_sim_agents.ipynb
   14  Scenario Gen: gerar cenários do zero ................. tutorial_scenario_gen.ipynb
        │
NÍVEL 5 ─ Síntese
   15  Direção end-to-end baseada em visão .................. tutorial_vision_based_e2e_driving.ipynb
```

### Por que esta ordem?

- **Nível 1 antes de tudo**: todo tutorial posterior assume que você sabe ler um
  `Frame`/`Scenario` e transformar coordenadas (sensor → veículo → mundo →
  imagem). É o equivalente a aprender a carregar o ImageNet antes de treinar uma
  ResNet.
- **Nível 2 vai do familiar ao exótico**: detecção (você já conhece de 2D) →
  segmentação 3D (pontos em vez de pixels) → panóptica com rastreamento
  (adiciona tempo e múltiplas câmeras) → pose (estrutura fina) → reconstrução
  (geração/NeRF).
- **O documento 09 é a virada conceitual**: saímos de *pixels e pontos* para
  *agentes e mapas*. É o momento de entender por que existe um segundo dataset.
- **Nível 3** introduz o tempo futuro e a incerteza (múltiplos futuros
  possíveis). **Nível 4** fecha a malha: as previsões passam a *afetar* o mundo
  simulado. **Nível 5** junta tudo num único modelo.

### Tabela-resumo

| # | Guia | Notebook | Dataset | Problema real | Técnica central | Métrica principal | Dificuldade |
|---|---|---|---|---|---|---|---|
| 00 | [Panorama](00-panorama-veiculos-autonomos.md) | — | todos | Entender a pilha AV | — | — | ★☆☆☆☆ |
| 01 | [Primeiro contato](01-primeiro-contato-sensores.md) | `tutorial.ipynb`, `tutorial_local.ipynb` | Perception v1 | Ler e fundir sensores | Geometria de câmera, range images | mAP / APH (CLI) | ★☆☆☆☆ |
| 02 | [Formato v2](02-formato-v2-parquet.md) | `tutorial_v2.ipynb` | Perception v2 | Montar datasets de treino escaláveis | Joins relacionais (Dask) | — | ★★☆☆☆ |
| 03 | [Mapas HD](03-mapas-hd-e-coordenadas.md) | `tutorial_maps.ipynb` | Perception v1 + mapa | Localização e contexto viário | Transformações rígidas, poses | — | ★★☆☆☆ |
| 04 | [Camera-only 3D](04-deteccao-3d-camera-only.md) | `tutorial_camera_only.ipynb` | Perception v1 | Detectar em 3D sem LiDAR | Detecção monocular/multivista | LET-3D-AP/APL | ★★★☆☆ |
| 05 | [Semseg 3D](05-segmentacao-semantica-3d.md) | `tutorial_3d_semseg.ipynb` | Perception v1 | Entender cada ponto da cena | Segmentação de point clouds | mIoU | ★★★☆☆ |
| 06 | [PVPS 2D](06-segmentacao-panoptica-video-2d.md) | `tutorial_2d_pvps.ipynb` | Perception v1/v2 | Identidade consistente 360° no tempo | Panóptica + tracking | wSTQ | ★★★★☆ |
| 07 | [Keypoints](07-pose-humana-keypoints.md) | `tutorial_keypoints.ipynb` | Perception v1 | Ler intenção de pedestres | Pose estimation 2D/3D | OKS-AP, PCK, MPJPE | ★★★☆☆ |
| 08 | [Object assets](08-object-assets-reconstrucao.md) | `tutorial_object_asset.ipynb` | Perception v2 (assets) | Recriar objetos p/ simulação de sensores | NeRF, completion, raios | — (pesquisa) | ★★★★☆ |
| 09 | [Ponte](09-ponte-perception-motion.md) | — | Perception ↔ Motion | Por que dois datasets? | — | — | ★★☆☆☆ |
| 10 | [Motion](10-predicao-de-movimento.md) | `tutorial_motion.ipynb` | Motion | Prever trajetórias dos outros | Regressão multimodal | minADE, minFDE, MR, mAP | ★★★☆☆ |
| 11 | [Occupancy Flow](11-occupancy-flow.md) | `tutorial_occupancy_flow.ipynb` | Motion | Prever onde haverá alguém (até ocluídos) | Encoder-decoder CNN em BEV | AUC, Soft-IoU, EPE | ★★★★☆ |
| 12 | [WOMD multimodal](12-womd-multimodal-lidar-camera.md) | `tutorial_womd_lidar.ipynb`, `tutorial_womd_camera.ipynb` | Motion + sensores | Predição a partir de sensores | Fusão, tokens VQ-GAN | (as do Motion) | ★★★☆☆ |
| 13 | [Sim Agents](13-sim-agents.md) | `tutorial_sim_agents.ipynb` | Motion | Testar o AV em simulação realista | Políticas generativas em malha fechada | Realism metametric | ★★★★★ |
| 14 | [Scenario Gen](14-scenario-gen.md) | `tutorial_scenario_gen.ipynb` | Motion | Criar cenários de teste novos | Modelos generativos de cena | Realism metametric | ★★★★★ |
| 15 | [E2E Driving](15-e2e-driving.md) | `tutorial_vision_based_e2e_driving.ipynb` | End-to-End Driving | Dirigir direto das câmeras | Imitation learning end-to-end | RFS, ADE | ★★★★☆ |

### Trilhas alternativas

Se você tem um objetivo específico, não precisa fazer tudo:

| Objetivo | Trilha mínima |
|---|---|
| Visão computacional "clássica" aplicada a AV | 00 → 01 → 02 → 04 → 06 → 07 |
| Deep learning em nuvens de pontos 3D | 00 → 01 → 03 → 05 → 08 |
| Previsão de comportamento / séries temporais | 00 → 01 → 03 → 09 → 10 → 11 |
| Modelos generativos / simulação | 00 → 03 → 09 → 10 → 13 → 14 (+ 08) |
| Foundation models / end-to-end | 00 → 01 → 09 → 12 → 15 |

---

## 4. Antes de rodar os notebooks

- Instalação, Docker e Colab: veja [`../SETUP.md`](../SETUP.md).
- Onde baixar cada dataset e qual componente é necessário: [`../DATA_CATALOG.md`](../DATA_CATALOG.md).
- Termos técnicos: [`../GLOSSARY.md`](../GLOSSARY.md).
- Mapa do código-fonte: [`../ARCHITECTURE.md`](../ARCHITECTURE.md).

Quase todos os notebooks usam o pacote `waymo-open-dataset-tf-2-12-0==1.6.7` e
esperam que você **edite um caminho de arquivo** para apontar para dados
baixados do site da Waymo (exigem cadastro e aceite da licença não-comercial).
Os que rodam com dados já incluídos em [`tutorial/`](../../tutorial/) são
`tutorial.ipynb` e `tutorial_local.ipynb` (`frames`), `tutorial_maps.ipynb`
(`frames_with_maps.tfrecord`) e `tutorial_keypoints.ipynb`
(`frame_with_keypoints.tfrecord`). Os tutoriais WOMD LiDAR/câmera referenciam
arquivos `womd_*.tfrecord` que **não** estão no repositório — só o codebook
`womd_camera_codebook.npy` está; os demais vêm do download do Motion Dataset.
