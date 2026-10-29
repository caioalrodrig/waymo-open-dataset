# 00 — Panorama: a pilha de software de um veículo autônomo

> **Nível 0 · Leitura conceitual · ~20 min**
> Objetivo: ter um "mapa mental" do domínio antes de abrir qualquer notebook.

---

## 1. O problema em uma frase

Um veículo autônomo precisa, **10 vezes por segundo**, responder a quatro
perguntas:

1. **Onde estou?** (localização)
2. **O que existe ao meu redor?** (percepção)
3. **O que cada um desses agentes vai fazer?** (predição)
4. **O que eu devo fazer?** (planejamento → controle)

E a empresa que constrói esse veículo precisa responder uma quinta:

5. **Como provo que ele é seguro sem rodar bilhões de km na rua?** (simulação e avaliação)

O Waymo Open Dataset tem dados e código para estudar **cada uma dessas
perguntas** — por isso tem tantos tutoriais diferentes.

---

## 2. A pilha clássica ("modular")

```
                  ┌───────────────────────────── MUNDO REAL ─────────────────────────────┐
                  │                                                                        │
   Sensores ──▶ [PERCEPÇÃO] ──▶ [PREDIÇÃO] ──▶ [PLANEJAMENTO] ──▶ [CONTROLE] ──▶ volante/freio
   câmeras,      "o que há       "o que eles     "que trajetória    "como executar
   LiDAR,         ao redor?"      vão fazer?"      eu sigo?"          a trajetória?"
   radar             ▲                ▲                 ▲
                     │                │                 │
                 [MAPA HD + LOCALIZAÇÃO]  ── "onde estou, onde ficam as faixas, semáforos"

   ────────────── fora do carro ──────────────
   [SIMULAÇÃO]: re-executa logs reais com agentes simulados para testar o planejador
```

| Módulo | Entrada | Saída | Analogia para quem vem de CV/DL |
|---|---|---|---|
| **Percepção** | Pixels + nuvem de pontos | Caixas 3D, classes, segmentação, rastros (tracks) | Detecção/segmentação, mas em 3D, 360° e no tempo |
| **Predição** | Tracks passados + mapa | Distribuição de trajetórias futuras de cada agente | Previsão de séries temporais multimodal / geração condicionada |
| **Planejamento** | Predições + mapa + objetivo da rota | Trajetória do próprio carro (o *ego*) | Política de RL / imitation learning com restrições de segurança |
| **Controle** | Trajetória desejada | Ângulo de volante, aceleração | Controle clássico (PID, MPC) — fora do escopo deste repo |
| **Simulação** | Cena inicial real | Futuro simulado com agentes reativos | Modelo de mundo / modelo generativo de agentes |

### E a abordagem *end-to-end*?

Em vez de módulos separados, **um único modelo** vai dos sensores direto à
trajetória do ego. É a tendência atual (inspirada nos *foundation models*), e é o
foco do último tutorial ([15](15-e2e-driving.md)).

---

## 3. Onde cada dataset entra

```
                  PERCEPTION DATASET            MOTION DATASET (WOMD)          END-TO-END DRIVING
                 ┌──────────────────┐         ┌───────────────────────┐      ┌───────────────────┐
  Sensores ────▶ │ câmeras + LiDAR  │         │ (LiDAR/câmera só do   │      │ 8 câmeras 360°    │
                 │ + rótulos humanos│         │  1º segundo, opcional)│      │ + intenção do ego │
                 └────────┬─────────┘         └───────────┬───────────┘      └─────────┬─────────┘
                          │ treina                        │ treina                      │ treina
                          ▼                               ▼                             ▼
                     PERCEPÇÃO  ───── tracks + mapa ──▶  PREDIÇÃO  ──▶ SIMULAÇÃO     PERCEPÇÃO→PLANEJAMENTO
                  (tutoriais 01–08)                   (10–12)        (13–14)          num modelo só (15)
```

| Dataset | Escala | Unidade | O que traz | Pergunta que responde |
|---|---|---|---|---|
| **Perception** | ~1.150 segmentos | Segmento de 20 s a 10 Hz (~200 frames) | Imagens de 5 câmeras, 5 LiDARs, calibração, caixas 3D/2D humanas, segmentação, keypoints, mapas | *O que existe ao redor agora?* |
| **Motion (WOMD)** | 103.354 segmentos de 20 s, recortados em janelas de 9 s | Cenário de 9 s (1 s passado + 8 s futuro, 10 Hz) | Trajetórias de agentes (auto-rotuladas), mapa vetorial, semáforos; opcionalmente LiDAR/câmera do 1º segundo | *O que cada agente vai fazer?* |
| **End-to-End Driving** | 4.021 segmentos (2.037 treino / 479 validação / 1.505 teste) | Frame com 8 câmeras | Imagens, passado do ego (4 s), futuro do ego (5 s), intenção, notas de avaliadores humanos | *Qual trajetória eu devo seguir?* |

---

## 4. Vocabulário mínimo para não se perder

| Termo | Significado |
|---|---|
| **Ego / SDC / ADV / AV** | O próprio veículo autônomo (*self-driving car*, *autonomously driven vehicle*). |
| **Agente** | Qualquer outro participante do trânsito: veículo, pedestre, ciclista. |
| **Segmento / run segment / context** | Um trecho contínuo de gravação (no Perception, 20 s). Identificado por `context.name`. |
| **Frame** | Um "instante": todas as leituras de sensores sincronizadas em torno de um timestamp. |
| **Cenário (Scenario)** | No Motion, uma janela de 9 s com tracks + mapa. |
| **Track** | A sequência de estados (posição, velocidade, heading, caixa) de um mesmo objeto ao longo do tempo. |
| **Heading / yaw** | Para onde o objeto está apontando (ângulo em torno do eixo vertical). |
| **BEV (Bird's-Eye View)** | Vista de cima. Representação preferida para predição e planejamento: o trânsito é quase 2D. |
| **Mapa HD / roadgraph** | Mapa de alta definição em polilinhas: centros de faixa, bordas, faixas de pedestre, placas. |
| **Range image** | Formato nativo do LiDAR: uma "imagem" onde cada pixel é a distância medida por um feixe. |
| **Frames de coordenadas** | *Sensor* (relativo à câmera/LiDAR), *veículo* (origem no carro, x para frente), *mundo* (fixo na Terra). Converter entre eles é 50% do trabalho prático. |
| **Rótulo humano vs. auto-label** | Perception: caixas desenhadas por humanos. Motion: caixas produzidas por um sistema de percepção *offboard* (que pode olhar o passado e o futuro do log). |
| **Long tail** | Situações raras (obras, animais, acidentes, gestos de policiais) que dominam o risco real e são escassas nos dados. |
| **Malha aberta vs. malha fechada** | *Aberta*: o modelo prevê, mas o mundo segue o log gravado. *Fechada*: as ações do modelo alteram o mundo simulado (tutoriais 13–14). |

---

## 5. O que muda em relação à visão computacional que você conhece

1. **3D e métrico.** Não basta acertar o retângulo na imagem: o erro é medido
   em metros, e a orientação (heading) importa — um carro de lado e um de
   frente exigem reações opostas.
2. **360° e multi-sensor.** O mesmo objeto aparece em várias câmeras e no LiDAR
   em instantes ligeiramente diferentes (cada sensor tem seu próprio tempo de
   captura — ver [04](04-deteccao-3d-camera-only.md)).
3. **Temporal por natureza.** Identidades precisam ser consistentes entre
   frames (tracking) e o futuro é incerto e **multimodal** (o carro pode virar
   ou seguir reto).
4. **Interação.** O comportamento de um agente depende dos outros e do ego.
   Prever cada um isoladamente gera futuros fisicamente impossíveis (colisões).
5. **A métrica é desenhada para a segurança.** Muitas métricas do repo ponderam
   o que importa para dirigir: APH pune heading errado, LET tolera erro de
   profundidade em câmeras, Occupancy Flow avalia agentes ocluídos, Sim Agents
   pesa colisões e saídas de pista mais que velocidade.

---

## 6. Próximo passo

➡️ [01 — Primeiro contato com os sensores](01-primeiro-contato-sensores.md)
