# 06 — Segmentação panóptica panorâmica em vídeo (2D PVPS)

| | |
|---|---|
| **Notebook** | [`tutorial_2d_pvps.ipynb`](../../tutorial/tutorial_2d_pvps.ipynb) |
| **Dataset** | Perception v1 (`camera_segmentation_label` nas imagens) ou v2 (componente `camera_segmentation`) |
| **Nível** | 2 · ★★★★☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md), [02](02-formato-v2-parquet.md); segmentação semântica/instância |
| **Código-chave** | `utils/camera_segmentation_utils.py`, `wdl_limited/camera_segmentation/camera_segmentation_metrics.py`, deeplab2 (dependência externa) |
| **Arquivos auxiliares** | `tutorial/2d_pvps_{training,validation,test}_frames.txt` |

---

## 1. O problema no mundo real

Um carro com 5 câmeras vê o mesmo pedestre passar da câmera frontal-esquerda
para a lateral-esquerda enquanto ele atravessa. Para o sistema entender
**"é o mesmo pedestre"** e acompanhar seu movimento, a percepção precisa ser:

- **Panóptica**: classificar *todo* pixel (rua, céu, prédio — *stuff*) e separar
  cada instância de objeto contável (este carro, aquele carro — *things*);
- **Panorâmica**: consistente **entre câmeras** ao redor do veículo;
- **Em vídeo**: consistente **no tempo**, com o mesmo ID ao longo dos frames.

É a versão mais completa de "entender a cena em imagens", e a base para tracking
denso baseado em câmera.

---

## 2. Os dados

- **28 classes** de câmera (veículos, pedestres, ciclistas, placas, semáforos,
  postes, cones, vegetação, céu, estrada, calçada, …).
- Cada imagem rotulada traz um **rótulo panóptico** codificado como
  `classe × divisor + instância` (PNG), e um mapeamento de instâncias locais para
  **IDs globais** consistentes entre câmeras e no tempo.
- **Treino**: sequências de 5 instantes nos tempos
  `t = 0, 400, 600, 800, 1200 ms` × 5 câmeras = 25 imagens rotuladas. O
  espaçamento não uniforme mistura pares próximos (fáceis de associar) e
  distantes (deslocamentos grandes).
- **Validação**: segmentos inteiros a 5 Hz → 100 instantes × 5 câmeras = 500
  rótulos por sequência.

---

## 3. O que o notebook faz, passo a passo

1. (Opcional) instala uma versão mínima do **deeplab2**, usado nas métricas.
2. Lê frames com rótulos panópticos a partir do **proto `Frame` (v1)** e,
   alternativamente, dos **componentes Parquet (v2)** — mesma informação, dois
   caminhos.
3. Decodifica um rótulo panóptico e o separa em mapas **semântico** e de
   **instância**.
4. Lê várias câmeras × vários frames com `remap_to_global=True`: os IDs passam a
   ser **globais** (mesmo objeto = mesmo ID em qualquer câmera e instante).
5. Visualiza as 5 câmeras lado a lado (da esquerda para a direita) ao longo do
   tempo.
6. **Avalia** predições (aqui, um modelo "dummy" que prevê 0 em tudo) por
   sequência, de forma sequencial e com `multiprocessing`.
7. **Gera a submissão** do conjunto de teste (`CameraSegmentationFrame` por
   frame pedido no `.txt`) e empacota.

---

## 4. A técnica de IA

| Etapa | Técnicas típicas |
|---|---|
| Panóptica por imagem | Panoptic-DeepLab, Mask2Former, kMaX-DeepLab (cabeça semântica + cabeça de instância/"queries") |
| Consistência temporal | Video Panoptic Segmentation (VPSNet, ViP-DeepLab), propagação de queries entre frames |
| Consistência entre câmeras | Associação de instâncias nas zonas de sobreposição; projeção para um espaço comum (BEV ou panorama) |

> 💡 Pense em tracking-by-segmentation: você não associa caixas, associa
> *máscaras*, usando aparência + movimento + geometria entre câmeras.

---

## 5. Como se mede sucesso: wSTQ

O repositório calcula **STQ (Segmentation and Tracking Quality)** e sua versão
ponderada:

| Componente | Mede |
|---|---|
| **SQ / mIoU** | Qualidade da segmentação semântica por pixel |
| **AQ (Association Quality)** | Se os pixels de um mesmo objeto recebem o mesmo ID ao longo do tempo e entre câmeras |
| **STQ** | Média geométrica: `√(AQ × SQ)` |
| **wSTQ / wAQ** | Versões ponderadas para compensar a **sobreposição entre câmeras** (pixels que aparecem em duas câmeras não contam em dobro) |

A média geométrica força o modelo a ser bom nas duas coisas: segmentar bem mas
trocar IDs (ou o contrário) é punido.

---

## 6. Armadilhas comuns

- **`panoptic_label_divisor`** precisa ser maior que o número máximo de
  instâncias por frame — e deve bater com o `eval_config`.
- Esquecer `remap_to_global=True` faz IDs mudarem entre câmeras, destruindo o AQ.
- Imagens de câmeras laterais têm resolução diferente da frontal: use *padding*
  para concatenar (o notebook faz `_pad_to_common_shape`).
- A métrica é cara: paralelize por sequência.

---

## 7. Para ir além

- Treine um modelo panóptico por imagem e adicione uma associação simples entre
  frames por IoU de máscara; meça o ganho de AQ.
- Paper: *Waymo Open Dataset: Panoramic Video Panoptic Segmentation* (Mei et al.,
  ECCV 2022).

---

⬅️ [05 — Semseg 3D](05-segmentacao-semantica-3d.md) · ➡️ [07 — Pose humana](07-pose-humana-keypoints.md)
