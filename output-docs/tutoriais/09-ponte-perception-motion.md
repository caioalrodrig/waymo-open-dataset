# 09 — Ponte: do Perception Dataset ao Motion Dataset

> **Leitura conceitual · ~25 min · sem notebook**
> Responde: *por que existem dois datasets, e como o Motion complementa o
> Perception?*

---

## 1. A virada conceitual: de pixels para agentes

Até aqui (tutoriais 01–08) o objeto de estudo foram **sensores**: pixels,
pontos, caixas e máscaras. A pergunta era *"o que existe ao meu redor agora?"*.

A partir daqui (10–15) a pergunta muda para *"o que cada agente vai fazer nos
próximos segundos, e o que eu devo fazer?"*. Para responder, o modelo não
precisa (necessariamente) olhar pixels: precisa de uma representação
**intermediária** do mundo — quem está onde, a que velocidade, em que faixa,
com que semáforo.

```
     PERCEPTION DATASET                                  MOTION DATASET (WOMD)
 ┌──────────────────────────┐                     ┌────────────────────────────────┐
 │ imagens 5 câmeras        │                     │ tracks: estados de cada agente │
 │ range images 5 LiDARs    │   percepção         │   (x, y, z, l, w, h, heading,  │
 │ calibrações, poses       │ ───────────────▶    │    vx, vy, valid) × 91 passos  │
 │ rótulos humanos:         │  (o que um modelo   │ mapa vetorial (map.proto)      │
 │  caixas 3D/2D, semseg,   │   treinado nos      │ estados de semáforos           │
 │  panóptica, keypoints    │   tutoriais 01–08   │ quem prever (tracks_to_predict)│
 │                          │   produz)           │ [+ LiDAR/câmera do 1º segundo] │
 └──────────────────────────┘                     └────────────────────────────────┘
   "O que existe agora?"                             "O que vai acontecer?"
```

**A saída da percepção é a entrada da predição.** O Motion Dataset é,
essencialmente, o mundo "já percebido" — uma versão limpa do que um ótimo
sistema de percepção produziria — em escala muito maior.

---

## 2. Comparação lado a lado

| Aspecto | Perception | Motion (WOMD) |
|---|---|---|
| **Pergunta central** | O que existe ao redor? | Como os agentes vão se mover? |
| **Unidade de dado** | `Frame` (um instante com todos os sensores) | `Scenario` (janela de 9 s) |
| **Duração / frequência** | Segmentos de 20 s a 10 Hz | Segmentos de 20 s recortados em janelas de 9 s a 10 Hz: **1 s de passado (11 estados, incluindo o atual) + 8 s de futuro (80 estados)** |
| **Escala** | ~1.150 segmentos | **103.354 segmentos** (≈574 horas), em 6 cidades dos EUA |
| **Conteúdo principal** | Imagens, nuvens de pontos, calibração | Trajetórias, mapa vetorial, semáforos |
| **Origem dos rótulos** | Anotadores humanos | **Auto-rotulagem offboard**: um sistema de percepção que roda fora do carro e pode usar o log inteiro (passado e futuro), produzindo tracks de alta qualidade |
| **Seleção dos dados** | Diversidade de condições (dia/noite, clima, cidades) | **Garimpado por interações interessantes**: conversões não protegidas, *merges*, pedestres cruzando, ciclistas, etc. |
| **Sensores** | Completos, em todos os frames | Opcionais e só no 1º segundo: LiDAR (v1.2+) e tokens de câmera (v1.2.1+) |
| **Formato** | TFRecord de `Frame` (v1) / Parquet (v2) | TFRecord de `Scenario` (proto) ou de `tf.Example` (tensores de tamanho fixo) |
| **Tarefas/tutoriais** | Detecção, segmentação, pose, assets (01–08) | Predição de trajetórias, occupancy flow, sim agents, scenario gen (10–14) |

---

## 3. Por que o Perception sozinho não basta para aprender comportamento

1. **Volume de interações.** Comportamento é *long tail*: uma conversão à
   esquerda com pedestre cruzando e ciclista vindo em sentido contrário acontece
   poucas vezes por hora. Com ~1.150 segmentos de 20 s (~6 h de direção), há
   poucos exemplos de cada tipo de interação. O WOMD tem ~90× mais tempo
   de direção (≈574 h vs. ≈6,4 h), **escolhido** para conter interações.
2. **Horizonte temporal.** Prever 8 s exige contexto e futuro longos e contínuos;
   o WOMD é organizado exatamente em janelas `passado → futuro`.
3. **Custo de rotulagem.** Rotular caixas 3D à mão para centenas de horas é
   inviável. A auto-rotulagem offboard torna essa escala possível — e como ela
   "vê o futuro", produz tracks mais estáveis do que qualquer percepção a bordo.
4. **Separação de responsabilidades.** Com o mundo já abstraído, quem pesquisa
   predição não precisa resolver percepção primeiro, e os resultados ficam
   comparáveis (todos partem da mesma entrada).

## 4. Por que o Motion sozinho também não basta

1. **Não ensina a ver.** Sem pixels e pontos densos com rótulos humanos, não se
   treina um detector — e na vida real a predição recebe a saída *imperfeita* de
   um detector, não tracks perfeitos.
2. **Perde sinais visuais.** Pisca-alerta, seta ligada, olhar do pedestre, gesto
   do guarda: tudo isso some na abstração em caixas. Por isso a v1.2 adicionou
   LiDAR e câmera ao 1º segundo de cada cenário ([12](12-womd-multimodal-lidar-camera.md)).
3. **Auto-labels têm vieses próprios**: objetos muito ocluídos ou raros podem
   faltar ou ter classes erradas.

---

## 5. Os elos concretos entre os dois datasets

| Elo | Detalhe |
|---|---|
| **Mesmo `map.proto`** | Faixas, bordas, faixas de pedestre e placas usam a mesma definição. O tutorial [03](03-mapas-hd-e-coordenadas.md) já mostra o mapa no Perception. |
| **Mesmos tipos de objeto** | Veículo, pedestre, ciclista (+ outro), mesma definição de caixa 7-DoF e heading. |
| **Mesma frequência (10 Hz)** e mesmas convenções de coordenadas (frame do mundo; x/y/z em metros). |
| **Mesmo modelo de sensores** | O LiDAR do WOMD usa o mesmo formato de range image comprimida e calibração do Perception; as utilidades do tutorial [01](01-primeiro-contato-sensores.md) se aplicam. |
| **Mesma linhagem de métricas** | O código de métricas compartilha infraestrutura C++/TF (`metrics/`), e o mapa do Perception pode alimentar os modelos de predição. |

---

## 6. Como usar os dois juntos (pipelines de pesquisa)

```
(A) Pilha modular clássica
    Perception ─treina─▶ detector+tracker ─produz tracks─▶ modelo de predição ◀─treina─ Motion

(B) Robustez da predição a ruído de percepção
    Motion (tracks limpos) + ruído/oclusões simulados ─▶ preditor robusto
    validado com tracks gerados por um detector treinado no Perception

(C) Percepção + predição conjunta (end-to-end parcial)
    WOMD com LiDAR/câmera do 1º segundo ─▶ modelo que vai do sensor direto à trajetória futura
    (pré-treino do backbone de sensores com rótulos do Perception)

(D) Simulação
    Motion ─treina─▶ sim agents / scenario gen (13–14)
    Perception (object assets, tutorial 08) ─▶ renderização de sensores para a cena simulada
```

---

## 7. Que pergunta vai para qual dataset?

| Se a sua pergunta é… | Use |
|---|---|
| "Onde estão os objetos nesta imagem/nuvem?" | Perception |
| "Este pixel/ponto é calçada ou asfalto?" | Perception |
| "Este pedestre está virado para a rua?" | Perception (keypoints) |
| "Para onde este carro vai nos próximos 8 s?" | Motion |
| "Que regiões estarão ocupadas, inclusive por quem está atrás do ônibus?" | Motion (occupancy flow) |
| "Como os outros motoristas reagiriam se o meu carro freasse?" | Motion (sim agents) |
| "A seta ligada ajuda a prever a conversão?" | Motion + câmera (v1.2.1+) |
| "Qual trajetória eu deveria seguir vendo só as câmeras?" | End-to-End Driving |

---

## 8. Glossário específico do Motion (para os próximos tutoriais)

| Campo do `Scenario` | Significado |
|---|---|
| `tracks[]` | Lista de agentes; cada um com `object_type` e `states[91]` (posição, tamanho, heading, velocidade, `valid`) |
| `current_time_index` | Índice do "agora" (10, i.e., o 11º passo) |
| `sdc_track_index` | Qual track é o próprio AV |
| `tracks_to_predict` | Até 8 agentes que **devem** ser previstos e serão avaliados |
| `objects_of_interest` | Pares de agentes com interação relevante (usados no desafio interativo/conjunto) |
| `map_features` | Mapa vetorial (`map.proto`) |
| `dynamic_map_states[]` | Estado dos semáforos por passo de tempo |
| `compressed_frame_laser_data` | LiDAR do 1º segundo (precisa ser mesclado de outro arquivo — [12](12-womd-multimodal-lidar-camera.md)) |
| `frame_camera_tokens` | Tokens VQ-GAN das câmeras do 1º segundo ([12](12-womd-multimodal-lidar-camera.md)) |

---

⬅️ [08 — Object assets](08-object-assets-reconstrucao.md) · ➡️ [10 — Predição de movimento](10-predicao-de-movimento.md)
