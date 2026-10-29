# 11 — Occupancy & Flow: predição densa em grade BEV

| | |
|---|---|
| **Notebook** | [`tutorial_occupancy_flow.ipynb`](../../tutorial/tutorial_occupancy_flow.ipynb) |
| **Dataset** | Motion (WOMD), formato `tf.Example` + listas de IDs de validação/teste do desafio |
| **Nível** | 3 · ★★★★☆ |
| **Pré-requisitos** | [10](10-predicao-de-movimento.md); encoder-decoder CNN (U-Net); fluxo óptico ajuda |
| **Código-chave** | `utils/occupancy_flow_{data,grids,renderer,metrics,vis}.py`, `protos/occupancy_flow_metrics.proto`, `protos/occupancy_flow_submission.proto` |
| **Desafio original** | Occupancy and Flow Prediction (2022) |

---

## 1. O problema no mundo real

A predição por trajetórias (tutorial 10) tem três limitações práticas:

1. **Só prevê quem já foi detectado.** E o carro escondido atrás do caminhão
   estacionado, que pode aparecer a qualquer momento?
2. **Número de agentes variável** e custo crescente com cenas lotadas.
3. **Planejadores perguntam "esta região estará livre?"**, não "onde estará o
   agente nº 37?".

**Occupancy Flow** responde de forma densa, como uma imagem: para cada célula de
uma grade vista de cima (BEV) e para vários instantes futuros,

- **Ocupação**: probabilidade de haver um veículo ali;
- separada em **observados** (visíveis agora) e **ocluídos** (não visíveis agora,
  mas que aparecerão);
- **Fluxo**: o vetor de movimento `(dx, dy)` daquela célula — de onde veio o que
  está ali.

---

## 2. Os dados: de tracks para grades

O notebook usa as funções do repo para **rasterizar** o cenário. Configuração do
desafio (`OccupancyFlowTaskConfig`):

| Parâmetro | Valor | Significado |
|---|---|---|
| `grid_height/width_cells` | 256 × 256 | Resolução da grade |
| `pixels_per_meter` | 3,2 | → ~80 m × 80 m ao redor do ego |
| `sdc_x/y_in_grid` | (128, 192) | O ego fica centralizado lateralmente e mais para trás → mais área à frente |
| `normalize_sdc_yaw` | true | A grade gira com o ego (frente sempre "para cima") |
| `num_past_steps` / `num_future_steps` | 10 / 80 | Igual ao WOMD |
| `num_waypoints` | 8 | Uma predição por segundo futuro (1 s … 8 s) |
| `agent_points_per_side_length/width` | 48 / 16 | Cada caixa é amostrada como uma grade de pontos para pintar a ocupação |

**Entrada do modelo (23 canais):** roadgraph (1) + ocupação passada de veículos
(10) + ocupação atual de veículos (1) + ocupação passada de pedestres/ciclistas
(10) + atual (1).

**Saída do modelo (32 canais):** para cada um dos 8 waypoints, 4 canais —
ocupação observada (1), ocupação ocluída (1), fluxo `(dx, dy)` (2).

---

## 3. O que o notebook faz, passo a passo

1. Carrega e **visualiza** um cenário (mesma animação do tutorial 10).
2. Define o `OccupancyFlowTaskConfig`.
3. **Gera o ground truth** com as utilidades do repo:
   `occupancy_flow_grids.create_ground_truth_timestep_grids` →
   `create_ground_truth_waypoint_grids` → `create_ground_truth_vis_grids`.
4. Visualiza ocupação observada, ocluída e fluxo para cada um dos 8 waypoints.
5. **Modelo baseline**: encoder **ResNet50V2** (sem pesos pré-treinados) + decoder
   de 5 estágios `Conv → UpSampling → Conv` até a resolução original, com 32
   canais de saída.
6. **Perda**:
   - entropia cruzada sigmoide por pixel para ocupação observada e ocluída;
   - L1 no fluxo, só onde o fluxo de referência é não-nulo;
   - média sobre os waypoints.
7. Loop de treino curto, **inferência** e visualização das predições (incluindo
   "ocupação × fluxo" conjunta).
8. **Métricas** com `occupancy_flow_metrics.compute_occupancy_flow_metrics`.
9. **Submissão**: roda inferência no teste (filtrando pelos IDs do desafio),
   preenche `ChallengeSubmission` (ocupação e fluxo quantizados) e comprime.

---

## 4. A técnica de IA

Aqui você se sente em casa: é **predição densa imagem→imagem**, como
segmentação semântica ou estimativa de fluxo óptico.

```
 entrada BEV [256,256,23]  ──▶ ResNet50V2 ──▶ decoder (upsampling) ──▶ [256,256,32]
  mapa + ocupação passada         encoder                                 8 × (ocup. obs, ocup. ocl, dx, dy)
```

- **Ocupação** ≈ segmentação binária por pixel (sigmoid + cross-entropy).
- **Fluxo** ≈ fluxo óptico *reverso* (de onde veio cada pixel ocupado).
- **Consistência ocupação–fluxo**: o fluxo do waypoint *k* deve "transportar" a
  ocupação de *k−1* até *k*. A métrica *flow-warped* verifica isso.

Melhorias sugeridas pelo próprio notebook: trocar o encoder convolucional por um
encoder baseado em pontos (Transformer, PointPillars) alimentado com pontos
amostrados dos agentes com features ricas (`_sample_agent_points()` em
`occupancy_flow_renderer.py`). Referências: *Occupancy Flow Fields for Motion
Forecasting in Autonomous Driving* (Mahjourian et al., 2022) e *Scene
Transformer*.

---

## 5. Como se mede sucesso

Calculadas para veículos e médias sobre os 8 waypoints:

| Métrica | O que mede |
|---|---|
| **Observed AUC / Soft-IoU** | Qualidade da ocupação prevista para agentes visíveis agora |
| **Occluded AUC / Soft-IoU** | Idem para agentes **atualmente ocluídos** — a parte nova e difícil |
| **Flow EPE** (End-Point Error) | Erro médio do vetor de fluxo (em células), onde há fluxo real |
| **Flow-warped occupancy AUC / Soft-IoU** | Usa o fluxo previsto para deformar a ocupação do passo anterior e compara com a ocupação real — mede a **consistência** entre fluxo e ocupação |

AUC avalia a ordenação das probabilidades (independe de limiar); Soft-IoU avalia
a sobreposição sem binarizar.

---

## 6. Armadilhas comuns

- **Grade centrada e rotacionada pelo ego**: as predições precisam estar no mesmo
  frame que o GT gerado pelas utilidades — não reinvente a rasterização.
- **Waypoints não cumulativos** (`cumulative_waypoints: false`): cada waypoint é
  a ocupação naquele segundo, não a união até ali.
- **Fluxo reverso**: aponta do instante atual para o *anterior*. Inverter o sinal
  é um erro clássico.
- A perda de ocupação é muito pequena por pixel (grade quase vazia); o notebook
  soma e reescala antes de tirar a média para evitar gradientes minúsculos.
- Rodar a submissão inteira é lento (o notebook marca como "slow").

---

## 7. Para ir além

- Adicione *skip connections* (vire uma U-Net de verdade) e compare o Soft-IoU.
- Adicione estados de semáforos como canais extras de entrada.
- Analise onde aparecem as ocupações ocluídas previstas: elas deveriam surgir nas
  "sombras" de oclusão e em entradas de faixas.

---

⬅️ [10 — Predição de trajetórias](10-predicao-de-movimento.md) · ➡️ [12 — WOMD multimodal](12-womd-multimodal-lidar-camera.md)
