# 12 — WOMD multimodal: LiDAR e tokens de câmera no Motion Dataset

| | |
|---|---|
| **Notebooks** | [`tutorial_womd_lidar.ipynb`](../../tutorial/tutorial_womd_lidar.ipynb) · [`tutorial_womd_camera.ipynb`](../../tutorial/tutorial_womd_camera.ipynb) |
| **Dataset** | Motion v1.2+ (LiDAR) e v1.2.1+ (tokens de câmera), distribuídos em arquivos separados `{scenario_id}.tfrecord` |
| **Nível** | 3 · ★★★☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md) (range images), [09](09-ponte-perception-motion.md), [10](10-predicao-de-movimento.md) |
| **Código-chave** | `utils/womd_lidar_utils.py`, `utils/womd_camera_utils.py`, `protos/compressed_lidar.proto`, `tutorial/womd_camera_codebook.npy` |

---

## 1. O problema no mundo real

O tutorial [09](09-ponte-perception-motion.md) mostrou que a abstração em caixas
**perde sinais visuais** que humanos usam para prever comportamento:

- seta ou pisca-alerta ligados;
- portas de carro estacionado se abrindo;
- direção do olhar e postura de pedestres;
- gestos de ciclistas e agentes de trânsito;
- objetos que o sistema de percepção não rotulou (entulho, animais).

Além disso, a predição real recebe a saída de uma percepção **imperfeita**.
Ter sensores e trajetórias no mesmo cenário permite estudar modelos que vão
**do sensor direto à predição**, ou que usam o sensor para complementar as
caixas.

---

## 2. Os dados

| Modalidade | Campo no `Scenario` | Conteúdo | Cobertura temporal |
|---|---|---|---|
| **LiDAR** | `compressed_frame_laser_data[]` | Range images comprimidas dos 5 LiDARs + calibrações + pose, no mesmo formato do Perception | Só o histórico (passos 0…`current_time_index`, i.e., ~1 s) |
| **Câmera** | `frame_camera_tokens[]` | Para cada câmera e frame, **256 inteiros** (tokens) | Idem |

Os **tokens** são índices de um *codebook* de um **VQ-GAN** pré-treinado:
em vez de pixels, cada imagem é representada por 256 códigos discretos. O codebook
(`womd_camera_codebook.npy`) transforma cada token em um vetor de features
contínuo.

Os sensores **não vêm no arquivo principal** do cenário: estão em arquivos
`{scenario_id}.tfrecord` à parte, que precisam ser **mesclados**.

---

## 3. O que os notebooks fazem, passo a passo

### `tutorial_womd_lidar.ipynb`

1. Carrega um `Scenario` e lê seu `scenario_id`.
2. Localiza o arquivo de LiDAR correspondente (`{scenario_id}.tfrecord`), que é
   um `Scenario` contendo só `compressed_frame_laser_data`.
3. **Mescla** os dados no cenário original
   (`womd_lidar_utils.augment_womd_scenario_with_lidar_points`).
4. Para cada frame, extrai as calibrações, descomprime as range images e converte
   em **nuvem de pontos + features** (`extract_top_lidar_points`,
   `extract_side_lidar_points`), juntando os 5 LiDARs.
5. Mostra uma imagem de exemplo da nuvem.

### `tutorial_womd_camera.ipynb`

1. Mesmos passos 1–3, agora com `frame_camera_tokens`
   (`womd_camera_utils.add_camera_tokens_to_scenario`).
2. Carrega o codebook `.npy` e converte cada sequência de 256 tokens em
   **embeddings de câmera** (`get_camera_embedding_from_codebook`), por câmera e
   por frame.

---

## 4. A técnica de IA

| Ideia | Como os dados ajudam |
|---|---|
| **Fusão tardia** | Encoders separados (LiDAR → PointPillars/voxels; câmera → embeddings do codebook) cujas features são adicionadas aos tokens dos agentes num Transformer de cena (tutorial 10) |
| **Tokens como "palavras visuais"** | Com 256 tokens discretos por imagem, a câmera vira uma sequência que entra num Transformer junto com tokens de mapa e agentes — sem precisar treinar uma CNN de imagem |
| **Percepção + predição conjunta** | Usar o LiDAR do histórico para detectar e prever ao mesmo tempo, avaliando com as métricas do Motion |
| **Robustez** | Comparar predições feitas a partir de tracks perfeitos vs. a partir do sensor |

> 💡 **VQ-GAN em uma frase:** um autoencoder cujo espaço latente é quantizado
> para o vetor mais próximo de um dicionário aprendido (codebook). Você recebe
> os índices; o codebook devolve os vetores. É a mesma ideia usada para
> tokenizar imagens em modelos generativos multimodais.

---

## 5. Como se mede sucesso

Não há métrica nova: os sensores servem como **entrada adicional** e o modelo é
avaliado com as métricas do Motion (minADE, minFDE, miss rate, mAP —
[10](10-predicao-de-movimento.md)) ou do Occupancy Flow ([11](11-occupancy-flow.md)).
A pergunta científica é: *quanto os sensores melhoram a predição em relação a
usar só caixas?*

---

## 6. Armadilhas comuns

- **Sensores só no passado**: não existem para os 8 s futuros (seria
  vazamento do alvo).
- **Os arquivos `womd_*.tfrecord` referenciados nos notebooks não estão no
  repositório**; você precisa baixá-los do Motion Dataset (só o codebook está em
  `tutorial/`).
- Garanta que o `scenario_id` do arquivo de sensor bate com o do cenário antes de
  mesclar.
- Os tokens não podem ser decodificados de volta para pixels com o que o repo
  fornece — use-os como features.

---

## 7. Para ir além

- Some a média dos embeddings da câmera frontal ao encoder do agente-alvo no
  modelo do tutorial 10 e meça a variação do miss rate em conversões.
- Detecte, a partir do LiDAR do histórico, objetos que **não** estão em `tracks`
  e veja se eles influenciam o futuro dos agentes rotulados.

---

⬅️ [11 — Occupancy & Flow](11-occupancy-flow.md) · ➡️ [13 — Sim Agents](13-sim-agents.md)
