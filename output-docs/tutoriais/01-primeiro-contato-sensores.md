# 01 — Primeiro contato: câmeras, LiDAR e range images

| | |
|---|---|
| **Notebooks** | [`tutorial.ipynb`](../../tutorial/tutorial.ipynb) (Colab) · [`tutorial_local.ipynb`](../../tutorial/tutorial_local.ipynb) (mesmo conteúdo, kernel Docker local) |
| **Dataset** | Perception v1 (TFRecord de protos `Frame`) — 2 frames incluídos em `tutorial/frames` |
| **Nível** | 1 · ★☆☆☆☆ |
| **Pré-requisitos** | [00 — Panorama](00-panorama-veiculos-autonomos.md) |
| **Código-chave** | `utils/frame_utils.py`, `utils/range_image_utils.py`, `utils/transform_utils.py`, `dataset.proto`, `label.proto` |

---

## 1. O problema no mundo real

Antes de qualquer modelo, um AV precisa **ver**. A Waymo usa dois tipos de
sensor complementares:

| Sensor | Pontos fortes | Pontos fracos |
|---|---|---|
| **Câmera** | Cor, textura, semáforos, placas, alta resolução, barata | Sem profundidade direta; sofre com noite, ofuscamento |
| **LiDAR** | Distância precisa em metros, funciona no escuro, 3D nativo | Esparso a longa distância, sem cor, caro |

Nenhum sozinho basta. **Fusão sensorial** — saber qual pixel corresponde a qual
ponto 3D — é a base de quase todo sistema de percepção. Este tutorial ensina
exatamente essa mecânica.

---

## 2. Os dados: anatomia de um `Frame`

Cada arquivo `segment-*.tfrecord` é uma sequência de ~200 `Frame`s (20 s a
10 Hz). Um `Frame` contém:

```
Frame
├── context                  ← compartilhado por todo o segmento
│   ├── name                 (ID do segmento)
│   ├── camera_calibrations  (intrínsecos + extrínsecos de cada câmera)
│   ├── laser_calibrations   (extrínsecos + ângulos dos feixes de cada LiDAR)
│   └── stats                (hora do dia, clima, local, contagem de objetos)
├── timestamp_micros
├── pose                     ← transformação veículo → mundo (4x4)
├── images[]                 ← 5 câmeras: FRONT, FRONT_LEFT, FRONT_RIGHT, SIDE_LEFT, SIDE_RIGHT (JPEG)
├── lasers[]                 ← 5 LiDARs: TOP + FRONT, SIDE_LEFT, SIDE_RIGHT, REAR (range images comprimidas)
├── laser_labels[]           ← caixas 3D (7 graus de liberdade) + tipo + ID de track
└── camera_labels[]          ← caixas 2D por câmera
```

### O que é uma *range image*?

O LiDAR gira e dispara feixes em várias elevações. Em vez de salvar uma lista de
pontos (x, y, z), a Waymo salva uma **imagem** onde:

- **linha** = elevação do feixe (ex.: 64 linhas para o LiDAR TOP),
- **coluna** = azimute (ângulo horizontal),
- **canais** = alcance (distância), intensidade, *elongation* (alongamento do
  pulso) e flag de "No Label Zone".

E há **dois retornos** por feixe (*first* e *second return*): um mesmo pulso
pode bater numa folha semitransparente e depois na parede atrás.

> 💡 **Para quem vem de CNN:** a range image é um tensor 2D denso, então você
> pode aplicar convoluções direto nela (modelos como RangeNet, LaserNet fazem
> isso). Converter para nuvem de pontos permite usar PointNet, voxels etc. Cada
> representação tem seus *trade-offs*.

---

## 3. O que o notebook faz, passo a passo

1. **Lê um frame** do TFRecord e examina `frame.context` (calibrações, stats).
2. **Visualiza as 5 câmeras** com as caixas 2D (`camera_labels`) desenhadas.
3. **Visualiza as range images** (alcance, intensidade, elongation) do LiDAR TOP,
   para os dois retornos.
4. **Converte range images em nuvem de pontos 3D** no frame do veículo
   (`frame_utils.parse_range_image_and_camera_projection` +
   `convert_range_image_to_point_cloud`) e conta pontos por sensor.
5. **Projeta os pontos LiDAR em cada imagem**, colorindo por distância — o
   "olá mundo" da fusão câmera + LiDAR.
6. (Somente `tutorial.ipynb`) **Compila o repo com Bazel** e roda o binário C++
   `compute_detection_metrics_main` com predições/GT falsos, mostrando como as
   métricas de detecção oficiais são calculadas.

---

## 4. A técnica por trás: geometria de sensores

O passo 5 é a cadeia de transformações que você usará em todos os tutoriais:

```
ponto no frame do LiDAR
   │  extrínseco do LiDAR  (4x4)
   ▼
ponto no frame do VEÍCULO   (x para frente, y para a esquerda, z para cima)
   │  inverso do extrínseco da câmera
   ▼
ponto no frame da CÂMERA
   │  intrínseco (focal, centro óptico, distorção) + rolling shutter
   ▼
pixel (u, v) na imagem
```

O dataset já entrega a projeção pré-calculada (`camera_projections`), mas
entender a cadeia é essencial para os tutoriais [03](03-mapas-hd-e-coordenadas.md),
[04](04-deteccao-3d-camera-only.md) e [15](15-e2e-driving.md).

---

## 5. Como se mede sucesso: métricas de detecção 3D

O binário do passo 6 calcula as métricas usadas no desafio de detecção 3D:

| Métrica | O que mede | Por que existe |
|---|---|---|
| **AP** (Average Precision) | Área sob a curva precisão × revocação, com casamento por IoU 3D (0,7 para veículos; 0,5 para pedestres/ciclistas) | Padrão de detecção, agora em 3D |
| **APH** (AP ponderado por Heading) | Cada acerto vale `1 − |Δheading|/π` | Errar a direção de um carro é perigoso, mesmo com a caixa no lugar certo |
| **LEVEL_1 / LEVEL_2** | Estratos de dificuldade: L2 inclui objetos com poucos pontos LiDAR (≤ 5) | Separar "fácil" de "quase invisível" |
| **MOTA / MOTP** | Métricas de *tracking* (manter IDs no tempo) | Percepção precisa ser consistente entre frames |

---

## 6. Armadilhas comuns

- **Retornos e sensores**: a nuvem "completa" é a união dos 5 LiDARs e dos dois
  retornos. Esquecer o segundo retorno ou os LiDARs laterais muda muito a
  densidade perto do carro.
- **Range image tem pixels inválidos** (alcance ≤ 0 = feixe sem retorno). Filtre
  antes de converter.
- **O LiDAR TOP tem `beam_inclinations` não uniformes**: não suponha elevações
  igualmente espaçadas.
- **Caixas 2D (`camera_labels`) e 3D (`laser_labels`) são rotuladas
  separadamente**; a associação entre elas é explícita só para alguns tipos.
  O tutorial [04](04-deteccao-3d-camera-only.md) aprofunda isso.

---

## 7. Para ir além

- Desenhe as caixas 3D (`laser_labels`) sobre a nuvem de pontos e depois
  projete os 8 cantos de cada caixa nas imagens.
- Treine um detector de veículos em BEV: rasterize a nuvem de pontos numa grade
  vista de cima (altura máxima, densidade, intensidade) e use uma CNN 2D comum.
  É a ideia por trás de PointPillars.
- Paper do dataset: *Scalability in Perception for Autonomous Driving: Waymo
  Open Dataset* (Sun et al., CVPR 2020).

---

⬅️ [00 — Panorama](00-panorama-veiculos-autonomos.md) · ➡️ [02 — Formato v2 (Parquet)](02-formato-v2-parquet.md)
