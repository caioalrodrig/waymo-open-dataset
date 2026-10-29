# 08 — Object assets: reconstrução 3D e NeRF por objeto

| | |
|---|---|
| **Notebook** | [`tutorial_object_asset.ipynb`](../../tutorial/tutorial_object_asset.ipynb) |
| **Dataset** | Perception v2 — componentes `object_asset_*` + `lidar_box` (lidos do bucket GCS) |
| **Nível** | 2 · ★★★★☆ |
| **Pré-requisitos** | [02](02-formato-v2-parquet.md) (v2 e joins), [01](01-primeiro-contato-sensores.md) (câmeras/LiDAR); noções de NeRF ajudam |
| **Código-chave** | `v2/perception/object_asset.py`, `v2.ObjectAssetRayCodec`, biblioteca `visu3d` |

---

## 1. O problema no mundo real

Para validar um AV, simulação é indispensável: é impossível encontrar na rua,
sob demanda, "um caminhão de mudança parado em fila dupla na neblina".
Simuladores precisam renderizar **sensores realistas** (imagens, LiDAR) de
objetos que **não estavam** no log original ou que estavam em outra posição.

Isso exige **ativos 3D** de carros e pessoas reais, com aparência e geometria
fiéis — inclusive objetos raros (*long tail*). Fazer isso à mão não escala;
reconstruí-los a partir dos próprios dados de direção, sim. É também um passo
para **completar objetos ocluídos** (ver o carro inteiro a partir de visões
parciais).

---

## 2. Os dados

Um conjunto **centrado em objeto** (categorias `VEHICLE` e `PEDESTRIAN`),
extraído do Perception v2:

| Componente | Conteúdo |
|---|---|
| `ObjectAssetLiDARSensorComponent` | Pontos LiDAR dentro de cada caixa |
| `ObjectAssetCameraSensorComponent` | Recortes de imagem do objeto, configuração do recorte, pontos LiDAR projetados |
| `ObjectAssetRayComponent` | Para cada pixel do recorte: origem e direção do **raio de câmera** no frame da caixa |
| `ObjectAssetRayCompressedComponent` | Versão comprimida dos raios (erro ~1e-5 m) + `codec_config.json` |
| `ObjectAssetAutoLabelComponent` | Máscaras densas auto-rotuladas (segmentação) nos recortes |
| `ObjectAssetRefinedPoseComponent` | Pose de caixa refinada por registro de nuvens (só veículos) |

---

## 3. O que o notebook faz, passo a passo

1. Autentica no GCP e lê os componentes direto do bucket com `gcsfs` + Dask.
2. Lista os assets de um segmento para a categoria escolhida.
3. **Campos de câmera**: mostra os recortes do mesmo objeto vistos por várias
   câmeras (FRONT, FRONT_LEFT, …) e ao longo do tempo, com máscaras de
   segmentação aplicadas.
4. **Raios no frame do objeto**: desprojeta cada pixel em 3D e visualiza os raios
   de câmera — um frame e depois vários frames agregados. Para veículos (rígidos),
   todas as observações se sobrepõem num mesmo sistema centrado no objeto.
5. **LiDAR**: visualiza os pontos de cada observação e compara a agregação com a
   caixa original vs. a **caixa refinada** (o alinhamento melhora visivelmente).
6. **Raios comprimidos**: decodifica com `ObjectAssetRayCodec` e repete a
   visualização.

---

## 4. A técnica de IA

Este é o tutorial que mais se aproxima de **visão 3D generativa**:

| Tarefa | Ideia | Por que os dados ajudam |
|---|---|---|
| **NeRF / 3D Gaussian Splatting por objeto** | Aprender uma função de radiância a partir de raios (origem, direção) → cor | Os raios já vêm no frame do objeto; as máscaras separam objeto de fundo |
| **Completion de nuvens** | Dado um scan parcial, prever a forma completa | Várias observações do mesmo objeto, alinhadas pela pose refinada, dão uma "forma de referência" |
| **Ativos generativos** | Gerar carros/pessoas novos e plausíveis (difusão 3D, GANs) | Escala: milhares de objetos reais, com iluminação e oclusão reais |

> 💡 Se você já viu NeRF com fotos de um objeto num estúdio, aqui o desafio é o
> "mundo selvagem": poucas vistas, iluminação variável, oclusão, borrão de
> movimento e um objeto que pode se deformar (pedestres).

---

## 5. Como se mede sucesso

Não há desafio oficial ligado a este tutorial; é um recurso de pesquisa. Na
literatura, avalia-se com PSNR/SSIM/LPIPS em vistas reservadas (reconstrução),
Chamfer distance (completion) e FID em renderizações (geração).

---

## 6. Armadilhas comuns

- A **pose refinada só existe para veículos**; pedestres não são rígidos.
- Os raios estão no **frame da caixa**: para combinar com a cena, aplique a pose
  da caixa no frame do veículo.
- O componente de raios sem compressão é grande; prefira o comprimido.
- Exige acesso ao bucket GCS autenticado (fluxo pensado para Colab).

---

## 7. Para ir além

- Treine um NeRF pequeno (ex.: instant-ngp) para um único veículo usando raios e
  máscaras de todas as suas observações.
- Paper relacionado: *GINA-3D: Learning to Generate Implicit Neural Assets in
  the Wild* (Shen et al., CVPR 2023), que introduziu este conjunto de assets.

---

⬅️ [07 — Pose humana](07-pose-humana-keypoints.md) · ➡️ [09 — Ponte Perception ↔ Motion](09-ponte-perception-motion.md)
