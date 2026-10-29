# Guia de Setup — Waymo Open Dataset

Este guia cobre a instalação e configuração do ambiente para trabalhar com o Waymo Open Dataset.

## Pré-requisitos

| Ferramenta | Versão | Notas |
|-----------|--------|-------|
| **Python** | 3.10 | Versão exata requerida pelo Bazel toolchain |
| **Bazel** | 5.4.0 | Build system para C++ e protos |
| **TensorFlow** | 2.13 | Framework principal |
| **Git** | qualquer | Controle de versão |
| **gsutil** | qualquer | Download dos datasets (parte do Google Cloud SDK) |

### Sistema operacional

- **Linux** (recomendado): Ubuntu 20.04+ ou equivalente
- **macOS**: Funciona com limitações (sem GPU support nativo para TF)
- **Windows**: Recomenda-se usar WSL2 ou Docker

## Opção 1: Docker (Mais Simples)

A maneira mais rápida de começar, sem instalar dependências no sistema.

### 1. Construir a imagem

```bash
cd tutorial/
docker build -f cpu-jupyter.Dockerfile --tag=waymo_tutorial .
```

### 2. Executar o container

```bash
docker run -p 8888:8888 waymo_tutorial
```

### 3. Acessar

Abra `http://localhost:8888` no navegador e abra `tutorial_local.ipynb`.

## Opção 2: Setup Local

### 1. Instalar Python 3.10

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3.10 python3.10-venv python3.10-dev

# Ou via pyenv (recomendado para gerenciar versões)
pyenv install 3.10
pyenv local 3.10
```

### 2. Criar ambiente virtual

```bash
python3.10 -m venv .venv
source .venv/bin/activate
```

### 3. Instalar dependências Python

```bash
pip install --upgrade pip
pip install tensorflow==2.13
pip install waymo-open-dataset-tf-2-12-0
```

As dependências principais definidas em `src/waymo_open_dataset/requirements.in`:

| Pacote | Versão | Uso |
|--------|--------|-----|
| tensorflow | 2.13 | Framework base |
| tensorflow_probability | 0.21.0 | Distribuições probabilísticas |
| numpy | 1.23.5 | Arrays numéricos |
| pandas | 1.5.3 | DataFrames (API V2) |
| matplotlib | 3.6.1 | Visualização |
| plotly | 5.13.1 | Visualização interativa |
| jax / jaxlib | 0.4.13 | Computação numérica |
| pyarrow | 16.0.0 | Leitura de Parquet (V2) |
| dask | 2023.3.1 | Processamento paralelo |
| scikit-learn | 1.2.2 | Machine learning |
| scikit-image | 0.20.0 | Processamento de imagens |
| pillow | 9.2.0 | Manipulação de imagens |
| visu3d | 1.5.1 | Visualização 3D |

### 4. Instalar Bazel (para compilar métricas C++)

```bash
# Via Bazelisk (recomendado — gerencia a versão automaticamente)
npm install -g @bazel/bazelisk

# Ou download direto
wget https://github.com/bazelbuild/bazel/releases/download/5.4.0/bazel-5.4.0-linux-x86_64
chmod +x bazel-5.4.0-linux-x86_64
sudo mv bazel-5.4.0-linux-x86_64 /usr/local/bin/bazel
```

### 5. Compilar (opcional, para métricas C++)

```bash
cd src/
bazel build //waymo_open_dataset/metrics/...
```

### 6. Instalar Google Cloud SDK (para download dos dados)

```bash
# Ubuntu/Debian
sudo apt install google-cloud-sdk

# Ou via script oficial
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

## Validação do Setup

### Teste rápido com Python

```python
import tensorflow as tf
print(f"TensorFlow: {tf.__version__}")

from waymo_open_dataset.protos import scenario_pb2
print("Proto imports OK")

from waymo_open_dataset import v2
print(f"V2 API OK — {len(v2.ALL_COMPONENTS)} componentes disponíveis")
```

### Teste com notebook

```bash
cd tutorial/
jupyter notebook tutorial_local.ipynb
```

O `tutorial_local.ipynb` utiliza dados de exemplo incluídos no repositório (`tutorial/frames`), não requer download do dataset completo.

## Dados de Exemplo (incluídos no repositório)

O diretório `tutorial/` contém dados de exemplo para testes rápidos:

| Arquivo | Tamanho | Conteúdo |
|---------|---------|----------|
| `frames` | ~8.2 MB | Frames de exemplo do Perception dataset |
| `frame_with_keypoints.tfrecord` | ~8.5 MB | Frame com keypoints humanos |
| `frames_with_maps.tfrecord` | ~17.6 MB | Frames com dados de mapa |

## Problemas Comuns

### `protobuf version mismatch`
O projeto usa protobuf 3.21. Certifique-se de não ter versões conflitantes:
```bash
pip install protobuf==3.20.*
```

### `TensorFlow GPU não encontrado`
Para GPU, instale os drivers NVIDIA e `tensorflow[and-cuda]==2.13`:
```bash
pip install tensorflow[and-cuda]==2.13
```

### `Bazel out of memory`
Limite a memória do Bazel:
```bash
bazel build --local_ram_resources=4096 //waymo_open_dataset/metrics/...
```

### Conflito de versão do NumPy
TensorFlow 2.13 requer NumPy < 1.24:
```bash
pip install numpy==1.23.5
```
