# Arquitetura do Repositório Waymo Open Dataset

Este documento descreve a organização dos módulos, suas responsabilidades e o fluxo de dados entre componentes.

## Visão Geral

```
┌─────────────────────────────────────────────────────────────────┐
│                        waymo-open-dataset                       │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│   tutorial/  │    docs/     │    src/      │     .agents/       │
│  (Notebooks) │ (Documentos) │  (Código)    │   (Automação AI)   │
└──────┬───────┴──────────────┴──────┬───────┴────────────────────┘
       │                             │
       │    Importa e utiliza        │
       └─────────────────────────────┘
```

## Módulo Principal: `src/waymo_open_dataset/`

### Definições de Formato (Protocol Buffers)

Os protobuf files definem o schema dos dados. São o ponto de partida para entender qualquer dataset.

```
src/waymo_open_dataset/
├── dataset.proto              # Schema principal do Perception dataset
│                              # Define: Frame, CameraImage, RangeImage,
│                              # CameraCalibration, LaserCalibration, Context
├── label.proto                # Labels: bounding boxes (2D/3D), tipos de objeto,
│                              # dificuldade, keypoints, associações camera↔lidar
└── protos/
    ├── scenario.proto         # Schema do Motion dataset (Track, ObjectState)
    ├── map.proto              # Mapas vetoriais (lanes, sinais, limites)
    ├── end_to_end_driving_data.proto  # Schema do E2E Driving dataset
    ├── metrics.proto          # Configuração de métricas de avaliação
    ├── motion_metrics.proto   # Métricas específicas de motion prediction
    ├── sim_agents_metrics.proto       # Métricas de simulação de agentes
    ├── occupancy_flow_metrics.proto   # Métricas de ocupação e fluxo
    ├── keypoint.proto         # Keypoints humanos (pose estimation)
    ├── segmentation.proto     # Labels de segmentação semântica 3D
    ├── camera_segmentation.proto      # Labels de segmentação de câmera
    ├── compressed_lidar.proto         # Formato comprimido de LiDAR
    ├── camera_tokens.proto            # Tokens de câmera (VQ-GAN)
    ├── *_submission.proto     # Formatos de submissão para challenges
    └── *_metrics.proto        # Configurações de métricas por challenge
```

### Métricas de Avaliação (`metrics/`)

Implementações em C++ de alta performance para avaliação de modelos.

```
metrics/
├── detection_metrics.{cc,h}       # AP, AP_H para detecção 2D/3D
├── tracking_metrics.{cc,h}        # MOTA, MOTP para tracking
├── motion_metrics.{cc,h}          # minADE, minFDE, mAP para motion prediction
├── segmentation_metrics.{cc,h}    # mIoU para segmentação semântica
├── mot.{cc,h}                     # Multi-Object Tracking base
├── iou.{cc,h}                     # Intersection over Union (2D e 3D)
├── hungarian.{cc,h}               # Hungarian algorithm para matching
├── matcher.{cc,h}                 # Matching entre predições e ground truth
├── ops/                           # TensorFlow custom ops (wrappers Python)
├── python/                        # Bindings Python
└── tools/                         # Ferramentas CLI
```

### Utilitários Python (`utils/`)

Funções auxiliares para manipulação e visualização de dados.

```
utils/
├── frame_utils.py                 # Parsing de frames, extração de point clouds
├── range_image_utils.py           # Conversão range image → point cloud 3D
├── box_utils.py                   # Operações com bounding boxes
├── transform_utils.py             # Transformações de coordenadas
├── geometry_utils.py              # Geometria computacional
├── camera_segmentation_utils.py   # Segmentação de câmera
├── keypoint_data.py               # Processamento de keypoints
├── keypoint_draw.py               # Visualização de keypoints
├── occupancy_flow_*.py            # Pipeline completo de occupancy flow
├── plot_maps.py                   # Visualização de mapas
├── trajectory_utils.py            # Manipulação de trajetórias
├── womd_camera_utils.py           # Utilitários de câmera WOMD
├── womd_lidar_utils.py            # Utilitários de LiDAR WOMD
└── sim_agents/                    # Pipeline de simulação de agentes
```

### API V2 Modular (`v2/`)

Interface Pythonic para o dataset v2, baseada em DataFrames (Pandas/Dask).

```
v2/
├── __init__.py          # API pública: todos os componentes exportados
├── component.py         # Classe base Component (deserialização Parquet → Python)
├── column_types.py      # Tipos vetoriais: Vec2s, Vec3d, etc.
├── dataframe_utils.py   # merge() e utilitários de DataFrame
└── perception/
    ├── box.py           # CameraBoxComponent, LiDARBoxComponent
    ├── camera_image.py  # CameraImageComponent
    ├── context.py       # CameraCalibrationComponent, LiDARCalibrationComponent, StatsComponent
    ├── keypoints.py     # CameraHumanKeypointsComponent, LiDARHumanKeypointsComponent
    ├── lidar.py         # LiDARComponent, LiDARPoseComponent, LiDARCameraProjectionComponent
    ├── object_asset.py  # ObjectAsset*Component (NeRF, reconstrução 3D)
    ├── pose.py          # VehiclePoseComponent
    ├── segmentation.py  # Camera/LiDARSegmentationLabelComponent
    └── utils/           # lidar_utils, object_asset_utils, object_asset_codec
```

**22 componentes disponíveis** que podem ser carregados individualmente, permitindo download seletivo.

### Código com Licença Restrita (`wdl_limited/`)

Código sob licença BSD 3-clause + patente limitada, uso apenas com Waymo Open Dataset.

```
wdl_limited/
├── camera/                    # Processamento de câmera (projeções, undistortion)
├── camera_segmentation/       # Panoptic segmentation de câmera
└── sim_agents_metrics/        # Métricas de simulação de agentes
```

### Conversão de Dados (`data_conversion/`)

```
data_conversion/
└── scenario_conversion.{cc,h}  # Conversão de Scenario proto ↔ tf.Example
```

## Fluxo de Dados

```
                    ┌──────────────┐
                    │  Raw Dataset │
                    │  (TFRecord / │
                    │   Parquet)   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
     ┌────────────┐ ┌───────────┐ ┌──────────┐
     │ Perception │ │  Motion   │ │   E2E    │
     │  Dataset   │ │  Dataset  │ │ Driving  │
     └─────┬──────┘ └─────┬─────┘ └────┬─────┘
           │               │            │
     ┌─────▼──────┐  ┌─────▼─────┐  ┌──▼──────────┐
     │ dataset.   │  │ scenario. │  │ e2e_driving_ │
     │ proto      │  │ proto     │  │ data.proto   │
     │ label.proto│  │ map.proto │  └──────────────┘
     └─────┬──────┘  └─────┬─────┘
           │               │
     ┌─────▼───────────────▼─────┐
     │         utils/            │
     │  frame_utils              │
     │  range_image_utils        │
     │  box_utils                │
     │  trajectory_utils         │
     └───────────┬───────────────┘
                 │
     ┌───────────▼───────────────┐
     │        metrics/           │
     │  detection_metrics        │
     │  tracking_metrics         │
     │  motion_metrics           │
     │  segmentation_metrics     │
     └───────────┬───────────────┘
                 │
     ┌───────────▼───────────────┐
     │      tutorial/            │
     │  Jupyter Notebooks        │
     │  (visualização e          │
     │   experimentação)         │
     └───────────────────────────┘
```

## Build System

O projeto usa **Bazel 5.4.0** como build system.

- `src/WORKSPACE` — Configuração raiz com dependências externas
- `src/.bazelrc` — Flags de compilação
- Cada diretório possui um arquivo `BUILD` com targets

Dependências principais gerenciadas via Bazel:
- TensorFlow 2.13
- Protobuf 3.21.9
- Abseil C++
- Google Test
- DeepLab2 (segmentação)

## Sensores do Veículo

O veículo autônomo Waymo possui:

| Sensor | Quantidade | Nomes |
|--------|-----------|-------|
| **LiDAR** | 5 | TOP, FRONT, SIDE_LEFT, SIDE_RIGHT, REAR |
| **Câmera** | 5–8 | FRONT, FRONT_LEFT, FRONT_RIGHT, SIDE_LEFT, SIDE_RIGHT, REAR_LEFT, REAR, REAR_RIGHT |

Os dados de LiDAR são armazenados como **range images** (imagens de alcance) e podem ser convertidos para point clouds 3D via `range_image_utils.py`.

## Tipos de Objetos Rotulados

| Tipo | Perception | Motion |
|------|-----------|--------|
| Vehicle | ✅ | ✅ |
| Pedestrian | ✅ | ✅ |
| Cyclist | ✅ | ✅ |
| Sign | ✅ | — |
| Other | — | ✅ |
