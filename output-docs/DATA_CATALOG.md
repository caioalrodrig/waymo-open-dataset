# Catálogo de Dados — Waymo Open Dataset

Este documento mapeia os datasets disponíveis, seus formatos, conteúdos e instruções de acesso.

## Visão Geral

| Dataset | Versão Atual | Cenários/Segmentos | Formato Primário | Formato V2 |
|---------|-------------|-------------------|-----------------|------------|
| **Perception** | v2.0.1 / v1.4.3 | ~1.150 segmentos (20s cada) | TFRecord | Parquet (modular) |
| **Motion** | v1.3.0 | 103.354 segmentos (20s cada), recortados em cenários de 9s | TFRecord + Scenario Proto | — |
| **End-to-End Driving** | v1.0.0 | 2.037 runs (training) + 1.505 (test) + 479 (validation) | TFRecord | — |

## 1. Perception Dataset

### Conteúdo

Dados de sensores de alta resolução coletados por veículos autônomos, com labels para múltiplas tarefas.

| Componente | Descrição |
|-----------|-----------|
| **Camera images** | Imagens de 5–8 câmeras sincronizadas |
| **LiDAR range images** | Range images de 5 sensores LiDAR |
| **3D bounding boxes** | Labels 3D para veículos, pedestres, ciclistas e sinais |
| **2D bounding boxes** | Projeções dos labels 3D nas imagens de câmera |
| **3D segmentação semântica** | Labels por ponto LiDAR (23 classes) |
| **2D segmentação panóptica** | Segmentação em vídeo nas imagens de câmera |
| **Keypoints humanos** | Pose estimation 2D e 3D para pedestres |
| **Object assets** | Recortes multi-sensor por objeto (para NeRF / reconstrução 3D) |
| **Mapas 3D** | Polylines/polígonos de lanes, limites e sinais |
| **Calibração** | Parâmetros intrínsecos/extrínsecos de cada sensor |

### Formato V1 (TFRecord)

- Arquivo: `segment-{id}.tfrecord`
- Conteúdo: serialização do proto `Frame` (definido em `dataset.proto`)
- Cada frame contém todas as imagens, range images e labels sincronizados

### Formato V2 (Parquet — modular)

O V2 decompõe cada frame em **componentes independentes**, permitindo download seletivo.

| Tag do Componente | Classe Python | Conteúdo |
|-------------------|--------------|----------|
| `camera_image` | `CameraImageComponent` | Imagens de câmera |
| `camera_box` | `CameraBoxComponent` | Bounding boxes 2D |
| `camera_calibration` | `CameraCalibrationComponent` | Calibração de câmera |
| `camera_segmentation` | `CameraSegmentationLabelComponent` | Segmentação panóptica |
| `camera_hkp` | `CameraHumanKeypointsComponent` | Keypoints 2D |
| `lidar` | `LiDARComponent` | Dados de range image |
| `lidar_box` | `LiDARBoxComponent` | Bounding boxes 3D |
| `lidar_calibration` | `LiDARCalibrationComponent` | Calibração LiDAR |
| `lidar_camera_projection` | `LiDARCameraProjectionComponent` | Projeção LiDAR → câmera |
| `lidar_camera_synced_box` | `LiDARCameraSyncedBoxComponent` | Boxes sincronizados com câmera |
| `lidar_segmentation` | `LiDARSegmentationLabelComponent` | Segmentação semântica 3D |
| `lidar_hkp` | `LiDARHumanKeypointsComponent` | Keypoints 3D |
| `lidar_pose` | `LiDARPoseComponent` | Pose de cada ponto LiDAR |
| `projected_lidar_box` | `ProjectedLiDARBoxComponent` | Boxes 3D projetados em 2D |
| `camera_to_lidar_box_association` | `CameraToLiDARBoxAssociationComponent` | Associação câmera ↔ LiDAR |
| `vehicle_pose` | `VehiclePoseComponent` | Pose do veículo ego |
| `stats` | `StatsComponent` | Estatísticas do segmento |
| `object_asset_*` | `ObjectAsset*Component` | Assets de objetos (5 componentes) |

### Splits

| Split | Segmentos | Uso |
|-------|----------|-----|
| Training | ~798 | Treinamento |
| Validation | ~202 | Validação |
| Testing | ~150 | Avaliação final (sem labels públicos) |

## 2. Motion Dataset (WOMD)

### Conteúdo

Trajetórias de objetos e mapas vetoriais 3D. **Não** é derivado dos segmentos do
Perception Dataset: são outros segmentos de direção (≈574 h), muito mais
numerosos, selecionados por conterem interações interessantes. As trajetórias são
**auto-rotuladas** por um sistema de percepção *offboard* (que processa o log
inteiro, passado e futuro), não por anotadores humanos.

| Componente | Descrição |
|-----------|-----------|
| **Tracks** | Trajetórias de objetos (posição, velocidade, heading) ao longo de 9 segundos |
| **Map features** | Lanes, stop signs, crosswalks, speed bumps, limites de pista |
| **Dynamic map** | Estado dos semáforos ao longo do tempo |
| **LiDAR data** | Dados LiDAR do primeiro 1s de cada cenário (v1.2+) |
| **Camera tokens** | Tokens VQ-GAN de câmeras (v1.2.1+) |

### Formato

- **Scenario proto** (`scenario.proto`): formato nativo
- **tf.Example**: formato alternativo, convertível via `data_conversion/scenario_conversion`

### Estrutura temporal

Cada cenário de 9 segundos tem 91 timesteps a 10Hz:
- **1s de histórico**: 10 timesteps passados + 1 atual (`current_time_index = 10`)
- **8s de futuro**: 80 timesteps

O primeiro 1s também contém dados de LiDAR e câmera (desde v1.2+).

### Splits

| Split | Cenários (janelas de 9s) | Uso |
|-------|---------|-----|
| Training | ≈487.000 | Treinamento |
| Validation | ≈44.000 | Validação |
| Testing | ≈45.000 | Avaliação final (sem o futuro dos agentes) |

> Valores aproximados segundo o paper do WOMD (Ettinger et al., ICCV 2021);
> confira o site oficial para a versão que você baixar.

### Tipos de objetos

Definidos em `Track.ObjectType`: VEHICLE, PEDESTRIAN, CYCLIST, OTHER.

## 3. End-to-End Driving Dataset

### Conteúdo

Dataset para aprendizado de direção autônoma end-to-end.

| Componente | Descrição |
|-----------|-----------|
| **Camera images** | Imagens de múltiplas câmeras |
| **Camera calibration** | Parâmetros de calibração |
| **Ego intent** | Intenção de direção do veículo: `GO_STRAIGHT`, `GO_LEFT`, `GO_RIGHT` (ou `UNKNOWN`), conforme `EgoIntent` em `end_to_end_driving_data.proto` |
| **Past states** | Histórico de 4s a 4Hz (posição, velocidade, aceleração) |
| **Future states** | Futuro de 5s a 4Hz (posição alvo) |
| **Preference trajectories** | Trajetórias com scores humanos [0, 10] (subset da validação) |

### Formato

- Proto: `E2EDFrame` (definido em `end_to_end_driving_data.proto`)
- Formato de arquivo: TFRecord

### Splits

| Split | Runs |
|-------|------|
| Training | 2.037 |
| Validation | 479 (com rater feedback labels e scenario tags) |
| Testing | 1.505 |

## Como Acessar os Dados

### 1. Registro

1. Acesse [waymo.com/open](https://waymo.com/open/)
2. Crie uma conta e aceite os [termos de uso](https://waymo.com/open/terms/)
3. Os termos permitem apenas **uso não-comercial**

### 2. Download

Os dados estão disponíveis via Google Cloud Storage:

```bash
# Perception v2 (modular — recomendado para download seletivo)
gsutil -m cp -r gs://waymo_open_dataset_v_2_0_1/ ./data/perception_v2/

# Perception v1 (TFRecord completo)
gsutil -m cp -r gs://waymo_open_dataset_v_1_4_3/ ./data/perception_v1/

# Motion
gsutil -m cp -r gs://waymo_open_dataset_motion_v_1_3_0/ ./data/motion/

# End-to-End Driving
gsutil -m cp -r gs://waymo_open_dataset_e2e_driving_v_1_0_0/ ./data/e2e/
```

> **Atenção:** Os datasets são grandes. O Perception v1 completo ocupa ~1.5 TB. O formato V2 permite download parcial por componente.

### 3. Download seletivo (V2 apenas)

```bash
# Apenas bounding boxes 3D do training set
gsutil -m cp -r gs://waymo_open_dataset_v_2_0_1/training/lidar_box/ ./data/lidar_box/

# Apenas imagens de câmera da validação
gsutil -m cp -r gs://waymo_open_dataset_v_2_0_1/validation/camera_image/ ./data/camera_image/
```

## Licenciamento dos Dados

O Waymo Open Dataset é licenciado sob o [Waymo Dataset License Agreement for Non-Commercial Use](https://waymo.com/open/terms/). Pontos importantes:

- ✅ Uso acadêmico e de pesquisa
- ✅ Publicação de resultados derivados
- ❌ Uso comercial
- ❌ Redistribuição dos dados brutos
