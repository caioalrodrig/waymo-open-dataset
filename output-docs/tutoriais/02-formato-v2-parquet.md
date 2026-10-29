# 02 — Formato v2 (Parquet) e engenharia de dados

| | |
|---|---|
| **Notebook** | [`tutorial_v2.ipynb`](../../tutorial/tutorial_v2.ipynb) |
| **Dataset** | Perception v2 (Apache Parquet, um arquivo por componente por segmento) |
| **Nível** | 1 · ★★☆☆☆ |
| **Pré-requisitos** | [01](01-primeiro-contato-sensores.md); noções de Pandas/SQL (joins) |
| **Código-chave** | `v2/__init__.py`, `v2/dataframe_utils.py` (`v2.merge`), `v2/perception/*.py` |

---

## 1. O problema no mundo real

O tutorial 01 lê um `Frame` monolítico: para obter só as caixas 3D, você baixa
também todas as imagens e range images. Em escala (≈1,5 TB no v1), isso é
proibitivo.

Em equipes reais de AV, cada modelo consome uma **fatia diferente** dos dados:

- Um detector camera-only quer imagens + caixas, sem LiDAR.
- Um modelo de tracking quer apenas caixas 3D ao longo do tempo.
- Um modelo de pose quer recortes de pedestres + keypoints.

O formato v2 resolve isso tratando o dataset como um **banco de dados
relacional colunar**: cada tipo de informação é uma tabela separada que você
baixa e junta sob demanda.

---

## 2. Os dados: componentes e chaves

```
{dataset}/{componente}/{context_name}.parquet

waymo_open_dataset_v_2_0_1/training/
├── camera_image/      10023947602400723454_1120_000_1140_000.parquet
├── camera_box/        ...
├── lidar/             ...
├── lidar_box/         ...
├── camera_hkp/        ...  (keypoints humanos 2D)
└── ...                (≈20 componentes — lista completa no DATA_CATALOG)
```

- **Colunas-chave** começam com `key.` e têm o mesmo nome em todas as tabelas:
  `key.segment_context_name`, `key.frame_timestamp_micros`,
  `key.camera_name`, `key.laser_name`, `key.laser_object_id`, …
- **Colunas de dados** têm o prefixo do componente:
  `[CameraImageComponent].image`, `[LiDARBoxComponent].box.center.x`, …
- O conteúdo é o mesmo do v1.4.2; só a organização muda.

---

## 3. O que o notebook faz, passo a passo

1. **Setup**: define uma função `read(tag)` que abre um componente com
   `dask.dataframe.read_parquet` (leitura preguiçosa, maior que a memória).
2. **Exemplo básico**: junta `camera_image` + `camera_box` → cada linha tem uma
   imagem e todas as suas caixas.
3. **Joins em diferentes níveis de granularidade** — a parte mais importante:

   | Nível | Uma linha representa… | Exemplo do notebook | Caso de uso |
   |---|---|---|---|
   | **Cena/track** | Um objeto em todos os frames | Trajetória por objeto (`lidar_box` agrupado por `laser_object_id`) | Tracking, predição |
   | **Frame** | Um instante com todos os objetos | LiDAR + caixas LiDAR + imagens + caixas de câmera | Detecção multimodal |
   | **Frame (com opcionais)** | Idem, mas caixas de câmera podem faltar (`left_nullable=True`) | — | Não descartar frames sem rótulo 2D |
   | **Objeto** | Um objeto em um frame | Imagem + LiDAR + caixa + keypoints | Pose estimation, crops |

4. **Estrutura de pastas e leitura crua** com qualquer leitor Parquet.
5. **Lista todos os componentes e colunas**.
6. **API orientada a objetos**: `v2.CameraImageComponent.from_dict(row)` converte
   uma linha num *dataclass* tipado (`.image`, `.pose.transform`, …).
7. **Filtro + join estilo SQL**: só a câmera frontal (`camera_name == 1`) com
   *push-down* de filtro, e o atalho `v2.merge` que descobre sozinho as chaves
   comuns.

---

## 4. A "técnica": pipeline de dados como parte do modelo

Não há rede neural aqui, mas este tutorial é **tão importante quanto um de
modelagem**. Em AV, a maior parte do esforço de engenharia está em:

- **Selecionar** o subconjunto certo (só noite, só chuva, só pedestres próximos)
  — o componente `stats` permite filtrar segmentos por condição.
- **Alinhar no tempo** dados que chegam de sensores diferentes.
- **Escolher a granularidade**: uma amostra de treino é um frame? um objeto? um
  track inteiro? Essa decisão define o que o modelo pode aprender.

> 💡 **Analogia:** é como montar um `torch.utils.data.Dataset` para COCO, mas com
> ~20 tabelas que precisam de joins. `v2.merge` é o seu `collate_fn`.

---

## 5. Armadilhas comuns

- **Linhas duplicadas por join**: num join objeto × imagem, a mesma imagem
  aparece repetida para cada objeto. Isso infla memória e enviesa amostragem se
  você não agrupar.
- **Inner join descarta silenciosamente** frames sem um componente (ex.: sem
  keypoints). Use `left_nullable`/`right_nullable` conscientemente.
- **`groupby` antes do join** (`right_group=True`) muda a forma dos dados: as
  colunas viram listas.
- Use **Dask para o dataset inteiro** e Pandas só para um segmento que caiba na
  memória.

---

## 6. Para ir além

- Monte um DataFrame "1 linha = 1 pedestre visível na câmera frontal à noite"
  usando `stats`, `camera_box` e `camera_image`.
- Compare o tempo/volume de download de `lidar_box` sozinho com o segmento v1
  completo.

---

⬅️ [01 — Primeiro contato](01-primeiro-contato-sensores.md) · ➡️ [03 — Mapas HD e coordenadas](03-mapas-hd-e-coordenadas.md)
