# 05 — Segmentação semântica 3D de nuvens de pontos

| | |
|---|---|
| **Notebook** | [`tutorial_3d_semseg.ipynb`](../../tutorial/tutorial_3d_semseg.ipynb) |
| **Dataset** | Perception v1 (≥ 1.3) — `frame.lasers[].ri_return*.segmentation_label_compressed` (só um subconjunto dos frames é rotulado) |
| **Nível** | 2 · ★★★☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md) (range images); segmentação semântica 2D |
| **Código-chave** | `protos/segmentation.proto`, `protos/segmentation_metrics.proto`, `protos/segmentation_submission.proto` |
| **Arquivo auxiliar** | `tutorial/3d_semseg_test_set_frames.txt` (frames do teste que devem ser submetidos) |

---

## 1. O problema no mundo real

Caixas 3D descrevem bem **objetos** (carros, pedestres), mas a cena tem muito
mais coisa relevante que não cabe numa caixa:

- **Onde é chão dirigível?** asfalto vs. calçada vs. grama;
- **Estruturas** que delimitam o espaço: muros, postes, vegetação, construções;
- **Objetos pequenos e irregulares**: cones, placas, hidrantes, entulho;
- **Classes que mudam o comportamento**: ciclista vs. moto, pedestre vs.
  "pedestre sobre veículo".

Segmentação semântica 3D dá **uma classe para cada ponto LiDAR** — uma
compreensão densa da geometria ao redor, útil para espaço livre, localização e
detecção de obstáculos fora das classes de caixa.

---

## 2. Os dados

- Rótulos ficam **na própria range image**: um tensor `[H, W, 2]` com
  `(instance_id, semantic_class)` por pixel, para os dois retornos.
- **23 classes**, incluindo: car, truck, bus, other vehicle, motorcyclist,
  bicyclist, pedestrian, sign, traffic light, pole, construction cone, bicycle,
  motorcycle, building, vegetation, tree trunk, curb, road, lane marker, other
  ground, walkable, sidewalk (além de `undefined`).
- Rótulos disponíveis principalmente para o **LiDAR TOP** e apenas em alguns
  frames de cada segmento.

---

## 3. O que o notebook faz, passo a passo

1. Varre um segmento e guarda apenas frames com rótulos de segmentação.
2. Visualiza a **range image de rótulos** (classe e instância) para os dois
   retornos — como uma imagem de segmentação "desenrolada" em 360°.
3. Converte range image → nuvem de pontos **mantendo o rótulo de cada ponto**
   (`convert_range_image_to_point_cloud_labels`).
4. Mostra um exemplo de nuvem colorida por classe.
5. Gera uma **submissão "dummy"** (todos os pontos com a mesma classe):
   - comprime arrays com `zlib`;
   - mapeia índices válidos da range image;
   - monta `SegmentationFrame` → `SegmentationFrameList` → arquivo de submissão,
     para validação e teste (usando a lista de frames do `.txt`).

---

## 4. A técnica de IA

A mesma tarefa da segmentação de imagens, mas com dados **esparsos,
irregulares e sem grade fixa**. Escolhas de representação:

| Representação | Arquitetura típica | Prós / contras |
|---|---|---|
| **Range image** (2D) | CNN 2D tipo U-Net (RangeNet++, SalsaNext) | Rápido, reaproveita CV 2D; bordas de objetos "vazam" entre linhas do feixe |
| **Pontos crus** | PointNet++, KPConv, Point Transformer | Fiel à geometria; caro para ~150 mil pontos |
| **Voxels esparsos** | Convoluções esparsas 3D (MinkowskiNet, Cylinder3D) | Estado da arte por muito tempo; exige bibliotecas especiais |
| **Multi-view** (range + BEV) | Fusão das duas vistas | Combina vantagens |

> 💡 Para quem conhece U-Net: a versão em range image é literalmente uma U-Net
> sobre um tensor `[64, 2650, C]`. Comece por ela.

---

## 5. Como se mede sucesso

**mIoU** (média do IoU por classe) sobre os pontos rotulados. A média por classe
impede que as classes frequentes (road, vegetation, building) escondam um
desempenho ruim nas raras (cones, motociclistas) — que costumam ser as mais
críticas para segurança.

---

## 6. Armadilhas comuns

- **Nem todo frame tem rótulo** — filtre antes de montar o dataset de treino.
- **Desbalanceamento extremo** entre classes: use perda ponderada, *focal loss*
  ou amostragem.
- A submissão é **no espaço da range image**, não numa lista de pontos: você
  precisa saber mapear sua predição de volta aos pixels válidos.
- O teste exige exatamente os frames listados em `3d_semseg_test_set_frames.txt`.

---

## 7. Para ir além

- Treine uma U-Net sobre a range image (canais: range, intensidade, elongation)
  e compare o IoU por classe com um modelo de voxels.
- Use `instance_id` para transformar a tarefa em **segmentação panóptica 3D**.

---

⬅️ [04 — Camera-only 3D](04-deteccao-3d-camera-only.md) · ➡️ [06 — Panóptica em vídeo 2D](06-segmentacao-panoptica-video-2d.md)
