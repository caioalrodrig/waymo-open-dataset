# 04 — Detecção 3D só com câmera (camera-synced boxes e métricas LET)

| | |
|---|---|
| **Notebook** | [`tutorial_camera_only.ipynb`](../../tutorial/tutorial_camera_only.ipynb) |
| **Dataset** | Perception v1 (≥ 1.4) — campos `camera_synced_box`, `most_visible_camera`, `projected_lidar_labels` |
| **Nível** | 2 · ★★★☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md) (cadeia de projeção), detecção 2D |
| **Código-chave** | `camera/ops/py_camera_model_ops.py`, `metrics/ops/py_metrics_ops.py`, `protos/metrics.proto` (`LongitudinalErrorTolerantConfig`) |
| **Desafio original** | 3D Camera-Only Detection (2022) |

---

## 1. O problema no mundo real

LiDAR é caro. Muitos fabricantes querem estimar posições 3D de objetos **usando
só câmeras**. O desafio é que uma imagem perde a profundidade: um carro pequeno
perto e um grande longe podem ocupar os mesmos pixels.

Há ainda um problema sutil de **sincronização**: o LiDAR gira (~100 ms por
volta) e as câmeras têm **rolling shutter** (as linhas da imagem são expostas em
sequência). Enquanto isso, o ego e os outros carros se movem. Uma caixa 3D
rotulada no instante em que o *LiDAR* viu o objeto não está exatamente onde o
objeto estava quando a *câmera* o fotografou. Para quem treina com imagens, isso
é ruído no rótulo.

---

## 2. Os dados

| Campo | O que é | Quando usar |
|---|---|---|
| `laser_labels[].box` | Caixa 3D sincronizada com o **LiDAR** | Modelos LiDAR |
| `laser_labels[].camera_synced_box` | A mesma caixa, deslocada para o instante em que a **câmera** observou o objeto | Modelos camera-only (alvo de treino) |
| `laser_labels[].most_visible_camera_name` | Câmera em que o objeto aparece melhor | Atribuir cada objeto a uma imagem |
| `projected_lidar_labels` | Caixas 3D (LiDAR-synced) já projetadas em 2D em cada câmera, considerando rolling shutter | Quando você precisa de caixas 2D coerentes com o 3D |
| `camera_labels` | Caixas 2D desenhadas diretamente na imagem | Detecção 2D pura |

Só objetos visíveis em alguma câmera têm `camera_synced_box` (as câmeras cobrem
~230° na horizontal, o LiDAR cobre 360°).

---

## 3. O que o notebook faz, passo a passo

1. Lê um frame e mostra as 5 câmeras.
2. Inspeciona as caixas 3D: compara `box` vs. `camera_synced_box` e
   `most_visible_camera`.
3. Desenha `projected_lidar_labels` (verde) em cada imagem.
4. Implementa `project_vehicle_to_image` com `py_camera_model_ops` e projeta os
   8 cantos de cada `camera_synced_box` → desenha **wireframes 3D** e caixas 2D
   (amarelo).
5. Compara as duas projeções e explica as pequenas diferenças (rolling shutter).
6. Monta uma **submissão** (`submission_pb2.Submission` com `metrics_pb2.Object`s).
7. Calcula **métricas LET** em Python com `py_metrics_ops.detection_metrics` e um
   `Config` com casamento húngaro e *breakdowns* por tipo de objeto, câmera e
   faixa de distância.

---

## 4. A técnica de IA

Detecção 3D monocular/multicâmera estima, para cada objeto, `(x, y, z, l, w, h,
heading)` a partir de pixels. Famílias de abordagens:

| Abordagem | Ideia | Exemplos |
|---|---|---|
| **Monocular direta** | Detector 2D + cabeças que regridem profundidade, dimensões e orientação | FCOS3D, SMOKE |
| **Pseudo-LiDAR** | Estima um mapa de profundidade e o converte em nuvem de pontos | Pseudo-LiDAR, DD3D |
| **Lift-splat para BEV** | "Levanta" features de imagem para 3D usando uma distribuição de profundidade e as projeta numa grade BEV | LSS, BEVDet, BEVFormer |

> 💡 O dataset dá a você o que esses modelos precisam: calibração exata de cada
> câmera, poses e rótulos 3D já sincronizados com a câmera.

---

## 5. Como se mede sucesso: métricas LET

Câmeras erram **profundidade** muito mais do que posição lateral. Com IoU 3D
padrão, um carro detectado no lugar certo mas 2 m mais longe a 50 m é um erro
total — o que torna todos os métodos camera-only quase indistinguíveis.

A família **LET (Longitudinal Error Tolerant)** relaxa isso:

| Métrica | Definição resumida |
|---|---|
| **LET-3D-AP** | AP em que o erro **ao longo da linha de visada** da câmera é tolerado até uma fração da distância (≈10%) antes de alinhar e calcular o IoU |
| **LET-3D-APL** | Cada acerto é ponderado pela *afinidade longitudinal* (quanto menor o erro de profundidade, maior o peso) — é a métrica principal do desafio |
| **LET-3D-APH** | Idem, ponderando pelo erro de heading |

Por que faz sentido para AV: para planejar, saber que há um carro "na sua faixa
a ~50 m" já é útil; a precisão de profundidade importa cada vez mais à medida que
o objeto se aproxima — e a tolerância proporcional à distância captura isso.

---

## 6. Armadilhas comuns

- **Usar `box` (LiDAR-synced) como alvo de um modelo de câmera** introduz erro
  sistemático em objetos rápidos.
- **Projeção com global shutter vs. rolling shutter**: as `camera_synced_box` já
  incorporam o rolling shutter, então projete-as com modelo global shutter (como
  o notebook faz).
- Exigir que **todos os 8 cantos** projetem com sucesso descarta objetos
  parcialmente visíveis (o notebook avisa).
- Os *timestamps* na submissão devem bater exatamente com `frame.timestamp_micros`.

---

## 7. Para ir além

- Treine uma cabeça de profundidade sobre um backbone 2D pré-treinado usando
  `camera_synced_box.center` como alvo, e meça LET-3D-APL por faixa de distância.
- Compare APH e LET-3D-APH do mesmo modelo para ver o efeito da tolerância.
- Paper: *LET-3D-AP: Longitudinal Error Tolerant 3D Average Precision for
  Camera-Only 3D Detection* (Hung et al.).

---

⬅️ [03 — Mapas HD](03-mapas-hd-e-coordenadas.md) · ➡️ [05 — Segmentação semântica 3D](05-segmentacao-semantica-3d.md)
