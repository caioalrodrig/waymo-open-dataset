# 07 — Estimativa de pose humana 2D e 3D (keypoints)

| | |
|---|---|
| **Notebook** | [`tutorial_keypoints.ipynb`](../../tutorial/tutorial_keypoints.ipynb) |
| **Dataset** | Perception v1 (`laser_labels[].laser_keypoints`, `camera_labels[].camera_keypoints`) — exemplo incluído em `tutorial/frame_with_keypoints.tfrecord` |
| **Nível** | 2 · ★★★☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md); pose estimation (ex.: COCO keypoints) |
| **Código-chave** | `protos/keypoint.proto`, `utils/keypoint_data.py`, `utils/keypoint_draw.py`, `metrics/python/keypoint_metrics.py` |
| **Doc complementar** | [`docs/pose_estimation_metric.md`](../../docs/pose_estimation_metric.md) |

---

## 1. O problema no mundo real

Uma caixa diz *onde* está o pedestre. A **pose** diz *o que ele está prestes a
fazer*:

- corpo e cabeça virados para a rua → provável travessia;
- olhando para o celular → pode não ter visto o carro;
- braço estendido → ciclista sinalizando conversão, guarda de trânsito
  orientando o fluxo;
- postura agachada/sentada → criança, pessoa em cadeira de rodas.

Usuários vulneráveis (pedestres, ciclistas) são o maior risco em ambiente
urbano. Pose é um sinal antecipado de intenção, que alimenta a predição de
movimento.

---

## 2. Os dados

- Keypoints anatômicos: nariz, testa, centro da cabeça, ombros, cotovelos,
  pulsos, quadris, joelhos e tornozelos (esquerda/direita), definidos em
  `keypoint.proto`.
- **2D (câmera)**: `(u, v)` em pixels + flag de visibilidade/oclusão.
- **3D (LiDAR)**: `(x, y, z)` em metros no frame do veículo.
- Cada conjunto de keypoints está associado a uma caixa (2D ou 3D) e a um objeto
  rastreado.

---

## 3. O que o notebook faz, passo a passo

1. Carrega um `Frame` com keypoints e agrupa os rótulos por objeto
   (`keypoint_data`).
2. Escolhe um pedestre e uma câmera.
3. **Keypoints na câmera**: desenha o esqueleto 2D sobre o recorte da imagem.
4. **Keypoints no LiDAR**: seleciona os pontos dentro da caixa do pedestre
   (`box_utils`) e plota em 3D (Plotly) os pontos + esqueleto 3D.
5. **Métricas**: calcula, para câmera e LiDAR, o conjunto combinado
   (`keypoint_metrics.create_combined_metric`) e mostra como usar métricas
   individuais (ex.: AP a OKS 0,95).
6. **Submissão**: um "detector falso" gera pedestres com keypoints aleatórios →
   `PoseEstimationSubmission` → `.tar.gz`.

---

## 4. A técnica de IA

| Modalidade | Abordagens típicas |
|---|---|
| **2D** | Top-down (detecta a pessoa, depois regride heatmaps por keypoint: HRNet, ViTPose) ou bottom-up (OpenPose) |
| **3D a partir de LiDAR** | Pontos dentro da caixa → PointNet/Transformer regride coordenadas 3D por keypoint |
| **3D multimodal** | Heatmaps 2D da câmera + profundidade/geometria do LiDAR; *lifting* 2D→3D |

Desafio específico de AV: pedestres pequenos e distantes, oclusão por carros
estacionados e **poucos rótulos 3D** — por isso abordagens fracamente
supervisionadas (usar rótulos 2D para treinar 3D) são comuns.

---

## 5. Como se mede sucesso

| Métrica | O que mede |
|---|---|
| **OKS-AP** (Object Keypoint Similarity) | Análogo ao IoU para keypoints: distância normalizada pelo tamanho do objeto e por uma escala por tipo de keypoint (pulsos são mais difíceis que ombros). AP sobre vários limiares de OKS. |
| **PCK** (Percentage of Correct Keypoints) | Fração de keypoints dentro de um raio de tolerância |
| **MPJPE** (Mean Per Joint Position Error) | Erro médio em metros/pixels por articulação |

> O notebook nota um detalhe: para amostras **sem** keypoints de referência, o
> OKS penaliza coordenadas fora da caixa, então o padrão é colocar keypoints
> ausentes no centro da caixa.

---

## 6. Armadilhas comuns

- Nem todo pedestre tem keypoints rotulados; nem todo keypoint é visível. Use as
  flags de visibilidade na perda.
- Keypoints 3D estão no frame do **veículo**; para normalizar a pose, leve-os ao
  frame da **caixa** (centrado e rotacionado pelo heading).
- As escalas por tipo de keypoint (`DEFAULT_PER_TYPE_SCALES`) mudam muito a OKS.

---

## 7. Para ir além

- Use a pose 3D como *feature* extra para prever se o pedestre vai atravessar
  nos próximos 2 s (une este tutorial com o [10](10-predicao-de-movimento.md)).
- Paper: *Multi-modal 3D Human Pose Estimation with 2D Weak Supervision in
  Autonomous Driving* (Zheng et al., Waymo).

---

⬅️ [06 — Panóptica 2D](06-segmentacao-panoptica-video-2d.md) · ➡️ [08 — Object assets](08-object-assets-reconstrucao.md)
