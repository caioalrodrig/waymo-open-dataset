# Glossário — Waymo Open Dataset & Cyber-Physical Systems

Termos técnicos utilizados no contexto do Waymo Open Dataset e da pesquisa em security de CPS.

---

## Sensores e Hardware

### LiDAR (Light Detection and Ranging)
Sensor que emite pulsos laser para medir a distância a objetos. Produz uma nuvem de pontos 3D (point cloud) representando a geometria do ambiente. O veículo Waymo possui 5 sensores LiDAR: TOP (360°), FRONT, SIDE_LEFT, SIDE_RIGHT e REAR.

### Range Image
Representação 2D dos dados LiDAR onde cada pixel contém a distância medida pelo sensor. É o formato nativo de armazenamento no Waymo Open Dataset. Pode ser convertido para point cloud 3D via `range_image_utils.py`.

### Point Cloud
Conjunto de pontos 3D (x, y, z) com atributos opcionais (intensidade, elongation). Gerado a partir das range images dos sensores LiDAR. Cada frame do dataset contém ~150.000–200.000 pontos.

### Rolling Shutter
Mecanismo de captura de câmera onde as linhas da imagem são expostas sequencialmente (não simultaneamente). Causa distorções em objetos em movimento. O dataset inclui a direção de leitura do rolling shutter para cada câmera.

### Calibração (Intrínseca e Extrínseca)
- **Intrínseca**: Parâmetros internos da câmera (distância focal, centro óptico, coeficientes de distorção)
- **Extrínseca**: Transformação 3D (rotação + translação) entre o frame do sensor e o frame do veículo

### Fusão Sensorial (Sensor Fusion)
Combinação de dados de múltiplos sensores (LiDAR + câmeras) para obter uma percepção mais robusta do ambiente. No dataset, a associação entre labels 2D (câmera) e 3D (LiDAR) é fornecida via `CameraToLiDARBoxAssociationComponent`.

---

## Labels e Anotações

### Bounding Box (3D / 2D)
Retângulo (2D) ou caixa (3D) que delimita um objeto no espaço. No formato 3D: centro (x, y, z), dimensões (comprimento, largura, altura) e heading (orientação). No formato 2D: coordenadas do retângulo na imagem.

### Heading
Ângulo (em radianos) que define a orientação frontal de um bounding box 3D. Medido no sentido anti-horário a partir do eixo X, normalizado para [-π, π).

### Difficulty Level
Nível de dificuldade de um label (LEVEL_1 = mais fácil, LEVEL_2 = mais difícil). Baseado no número de pontos LiDAR dentro do bounding box e na distância ao ego vehicle.

### No Label Zone (NLZ)
Áreas do cenário onde objetos não são rotulados por serem irrelevantes para a direção (ex: lado oposto de uma rodovia, estacionamentos distantes).

### Segmentação Semântica
Classificação de cada ponto LiDAR ou pixel de câmera em uma classe semântica (veículo, pedestre, vegetação, construção, etc.). O dataset v1.3+ inclui 23 classes semânticas para LiDAR.

### Segmentação Panóptica
Extensão da segmentação semântica que também distingue instâncias individuais de objetos (cada carro tem um ID único, não apenas a classe "veículo").

### Keypoints (Human Pose Estimation)
Pontos anatômicos de referência em pedestres (ombros, cotovelos, joelhos, etc.). Disponíveis em 2D (projetados na câmera) e 3D (no espaço LiDAR).

---

## Datasets e Formatos

### TFRecord
Formato binário do TensorFlow para armazenamento sequencial de dados serializados. Os dados são armazenados como `tf.train.Example` contendo features serializadas em Protocol Buffers.

### Protocol Buffers (Protobuf)
Framework de serialização de dados do Google. Define schemas (`.proto`) que são compilados para classes em múltiplas linguagens. O Waymo Open Dataset define ~31 arquivos `.proto`.

### Parquet
Formato colunar de armazenamento usado pelo V2 do dataset. Permite leitura seletiva de colunas e é compatível com Pandas/Dask DataFrames.

### Frame
Unidade temporal básica do Perception dataset. Contém todos os dados de sensores (imagens, range images) e labels correspondentes para um instante de tempo.

### Segmento (Segment)
Sequência de frames (~20 segundos) coletada durante uma viagem contínua. Identificado por um nome único. Contém calibração e contexto compartilhados entre frames.

### Cenário (Scenario)
Unidade do Motion dataset. Janela de 9 segundos contendo trajetórias de objetos e mapa vetorial. Cada cenário possui um veículo ego e objetos observados.

### Track
Sequência de estados de um objeto ao longo de um cenário no Motion dataset. Contém posição, velocidade e dimensões do bounding box em cada timestep.

---

## Métricas de Avaliação

### AP (Average Precision)
Métrica principal para detecção de objetos. Mede a área sob a curva precision-recall. Calculada por tipo de objeto e nível de dificuldade.

### APH (Average Precision Heading-weighted)
Variante do AP que penaliza erros na estimativa de heading (orientação) do bounding box.

### LET-3D-APL (Longitudinal Error Tolerant 3D AP)
Métrica para detecção 3D somente com câmera. Tolera erros longitudinais (profundidade) maiores que erros laterais, refletindo a dificuldade de estimação de profundidade com câmera.

### mAP (mean Average Precision)
Média do AP entre todos os tipos de objeto. Usado em motion prediction com bucketing por comportamento.

### minADE / minFDE
- **minADE** (minimum Average Displacement Error): Erro médio de deslocamento da melhor predição (entre K predições) ao longo do horizonte temporal
- **minFDE** (minimum Final Displacement Error): Erro de deslocamento no timestep final da melhor predição

### MOTA / MOTP
- **MOTA** (Multiple Object Tracking Accuracy): Combina false positives, false negatives e identity switches
- **MOTP** (Multiple Object Tracking Precision): Erro médio de localização de objetos corretamente trackados

### IoU (Intersection over Union)
Razão entre a área/volume de interseção e a área/volume de união de dois bounding boxes. Usado como threshold para matching entre predições e ground truth.

### mIoU (mean Intersection over Union)
Média do IoU por classe semântica. Métrica principal para segmentação semântica.

---

## Motion e Predição

### Occupancy Flow
Representação grid-based do ambiente futuro. Prediz quais células da grade estarão ocupadas e a direção do fluxo de objetos ao longo do tempo.

### Sim Agents
Simulação de agentes autônomos em cenários do dataset. Avalia a capacidade de um modelo de gerar trajetórias realistas para múltiplos agentes simultaneamente.

### Scenario Generation
Geração de cenários de teste para validação de sistemas de direção autônoma. Avalia diversidade e realismo dos cenários gerados.

---

## Cyber-Physical Systems (CPS)

### Cyber-Physical System (CPS)
Sistema que integra computação, comunicação e processos físicos. Veículos autônomos são CPS complexos onde falhas no componente "cyber" (software, algoritmos) têm consequências diretas no mundo físico.

### Adversarial Attack
Perturbação intencional de dados de entrada (imagens, point clouds) para enganar modelos de machine learning. No contexto de veículos autônomos: patches adversariais em imagens, spoofing de LiDAR, manipulação de sinais GPS.

### LiDAR Spoofing
Ataque que injeta pontos falsos no sensor LiDAR usando um laser externo. Pode criar objetos fantasma ou ocultar objetos reais da percepção do veículo.

### Adversarial Patch
Padrão visual físico (adesivo, cartaz) projetado para enganar classificadores de imagem. Pode fazer um sinal de PARE ser classificado como sinal de velocidade.

### Resiliência
Capacidade de um sistema CPS de manter operação segura sob ataques, falhas de sensor ou condições degradadas. Medida pela capacidade de detecção, mitigação e recuperação.

### Safety-Critical System
Sistema onde falhas podem causar danos a pessoas ou propriedade. Veículos autônomos são safety-critical e requerem validação rigorosa.

---

## Coordenadas e Transformações

### Vehicle Frame
Sistema de coordenadas centrado no veículo ego:
- **+X**: para frente
- **+Y**: para a esquerda
- **+Z**: para cima
- Origem: centro do eixo traseiro do veículo

### Global Frame
Sistema de coordenadas fixo no mundo. Usado para representar posições absolutas de objetos no Motion dataset.

### Ego Vehicle
O veículo autônomo que carrega os sensores e coleta os dados. Todos os dados são relativos ao ego vehicle.
