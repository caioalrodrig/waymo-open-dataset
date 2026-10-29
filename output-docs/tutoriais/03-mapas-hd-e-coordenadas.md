# 03 — Mapas HD e sistemas de coordenadas

| | |
|---|---|
| **Notebook** | [`tutorial_maps.ipynb`](../../tutorial/tutorial_maps.ipynb) |
| **Dataset** | Perception v1.4.2+ com mapa (`frame.map_features`) — exemplo incluído em `tutorial/frames_with_maps.tfrecord` |
| **Nível** | 1 · ★★☆☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md); álgebra de transformações rígidas (4x4) |
| **Código-chave** | `protos/map.proto`, `utils/plot_maps.py`, `utils/frame_utils.py` |

---

## 1. O problema no mundo real

Um AV não dirige "às cegas" sobre o asfalto: ele carrega um **mapa HD** com a
geometria exata de cada faixa, borda de pista, faixa de pedestre, lombada, placa
de pare e a conexão entre faixas (quem pode virar para onde).

O mapa resolve problemas que os sensores sozinhos não resolvem bem:

- **Contexto**: uma faixa pintada apagada continua no mapa.
- **Regras**: "essa faixa é só para virar à esquerda", "aqui há um pare".
- **Antecipação**: saber que há uma faixa de pedestre atrás do ônibus.
- **Predição**: carros quase sempre seguem centros de faixa — o mapa é o
  *prior* mais forte para prever trajetórias (tutoriais [10](10-predicao-de-movimento.md)+).

Para usar o mapa, porém, é preciso saber **onde o carro está nele** com precisão
centimétrica (localização) e expressar sensores e mapa no **mesmo sistema de
coordenadas**.

---

## 2. Os dados

`map.proto` define o mapa como **feições vetoriais** (polilinhas e polígonos),
não como imagem:

| Feição | Geometria | Atributos |
|---|---|---|
| `lane` | Polilinha do centro da faixa | Tipo (rodovia, rua, ciclovia), limite de velocidade, vizinhos esquerda/direita, faixas de entrada/saída, bordas |
| `road_line` | Polilinha | Tipo de pintura (tracejada branca, dupla amarela, …) |
| `road_edge` | Polilinha | Meio-fio ou mediana |
| `stop_sign` | Ponto | Faixas que ele controla |
| `crosswalk`, `speed_bump`, `driveway` | Polígono | — |

O mesmo `map.proto` é usado pelo **Motion Dataset** — esse é um dos elos entre os
dois datasets (ver [09](09-ponte-perception-motion.md)). Para a semântica de
vizinhos e bordas de faixa, veja [`docs/lane_neighbors_and_boundaries.md`](../../docs/lane_neighbors_and_boundaries.md).

---

## 3. O que o notebook faz, passo a passo

1. Carrega 2 frames de um segmento que inclui `map_features`.
2. Plota as feições do mapa em 3D (Plotly) com `plot_maps`.
3. Para cada frame:
   - converte range images → pontos no frame do **veículo**;
   - aplica `frame.pose` para levar os pontos ao frame do **mundo**;
   - aplica uma correção para o sistema de coordenadas do **primeiro frame**,
     que é onde o mapa está ancorado (`frame.map_pose_offset`).
4. Sobrepõe as nuvens de pontos ao mapa num gráfico interativo (girar, pan, zoom).

Resultado: você vê os pontos do LiDAR "caindo" sobre as faixas e meios-fios do
mapa — prova de que a cadeia de transformações está correta.

---

## 4. A técnica: transformações rígidas e poses

```
p_mundo  = T_mundo←veículo(t) · p_veículo
p_mapa   = (correção de offset do mapa) · p_mundo
```

- `T` é uma matriz 4x4 (rotação 3x3 + translação). Ela muda **a cada frame**
  porque o carro se move.
- Acumular frames no frame do mundo é como se constrói um "mapa de pontos"
  denso, e é o mesmo princípio usado para auto-rotular dados *offboard* (como os
  do Motion Dataset).
- Errar a ordem de multiplicação ou usar a pose do frame errado produz
  "fantasmas" (nuvens duplicadas e deslocadas) — um ótimo exercício de debug.

> 💡 **Para quem vem de CV:** pense em *homografias* generalizadas para 3D com
> corpos rígidos. Ferramentas que ajudam: `transform_utils.py`, `scipy.spatial.transform`.

---

## 5. Armadilhas comuns

- **Coordenadas do mapa ≠ mundo bruto**: sem o `map_pose_offset`, a nuvem não
  encaixa.
- **Pontos de objetos em movimento** não se alinham entre frames (carros
  "borram"). Isso é esperado — só a cena estática se alinha.
- Plotar muitos frames com Plotly fica lento; amostre pontos.

---

## 6. Para ir além

- Rasterize o mapa numa imagem BEV centrada no ego (canal por tipo de feição).
  Essa é exatamente a entrada que a CNN do tutorial [11](11-occupancy-flow.md) usa.
- Para cada caixa 3D de veículo, encontre a faixa mais próxima e calcule o
  desvio lateral — um *feature* clássico de predição.

---

⬅️ [02 — Formato v2](02-formato-v2-parquet.md) · ➡️ [04 — Detecção 3D só com câmera](04-deteccao-3d-camera-only.md)
