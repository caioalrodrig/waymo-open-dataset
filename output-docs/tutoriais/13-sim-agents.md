# 13 — Sim Agents: simular o tráfego em malha fechada

| | |
|---|---|
| **Notebook** | [`tutorial_sim_agents.ipynb`](../../tutorial/tutorial_sim_agents.ipynb) |
| **Dataset** | Motion (WOMD), formato `Scenario` (proto) — usa o split de **validação** para ter o futuro real |
| **Nível** | 4 · ★★★★★ |
| **Pré-requisitos** | [10](10-predicao-de-movimento.md); noções de modelos generativos e de RL (políticas, rollouts) |
| **Código-chave** | `utils/sim_agents/submission_specs.py`, `utils/sim_agents/visualizations.py`, `utils/trajectory_utils.py`, `wdl_limited/sim_agents_metrics/{metric_features,metrics}.py`, `challenge_2025_sim_agents_config.textproto`, `protos/sim_agents_submission.proto` |
| **Desafio original** | Sim Agents (2023–2025) |

---

## 1. O problema no mundo real

Para lançar uma nova versão do software de direção, é preciso testá-la em
milhões de situações. A rua é lenta, cara e arriscada; a solução é **simulação**.
Mas há um problema: se você re-executa um log gravado e o seu carro freia antes
do que o motorista original freou, o carro de trás no log **não reage** — ele
continua como gravado e "atravessa" o seu carro. O teste fica inválido.

É preciso substituir os outros agentes do log por **agentes simulados** (*sim
agents*) que:

- reajam ao AV e entre si (**malha fechada / closed-loop**);
- se comportem como **humanos reais** — nem perfeitos, nem caóticos;
- produzam **diversidade** (vários futuros plausíveis a partir da mesma cena).

É a diferença entre *prever* o futuro (tutorial 10) e *simular* um mundo onde
as suas próprias ações mudam o futuro.

---

## 2. A tarefa

Especificada em `submission_specs.py`:

| Item | Valor |
|---|---|
| Entrada | Cenário com histórico de 1 s (11 passos), mapa e semáforos |
| Simular | **Todos** os objetos válidos no passo atual (11º), incluindo o AV |
| Horizonte | **80 passos a 10 Hz (8 s)** |
| Rollouts | **32 simulações paralelas** por cenário |
| Saída por objeto | `x, y, z, heading` a cada passo |
| Avaliados | Os agentes de `tracks_to_predict` (mas todos participam das interações) |

**Regras de malha fechada:**

1. Simulação passo a passo: cada política só "vê" o mundo até o passo atual.
2. As políticas **não compartilham intenções futuras** — obrigatório entre o AV e
   os demais, porque no uso real o AV será trocado pelo planejador sendo testado.

---

## 3. O que o notebook faz, passo a passo

1. Carrega um `Scenario` da validação e plota os tracks.
2. Lê a configuração do desafio (número de passos, frequência, rollouts) e
   identifica os objetos a simular (`get_sim_agent_ids`).
3. **Política trivial**: extrapolação linear com velocidade constante + ruído
   aleatório, rodada **em lote para os 32 rollouts**, passo a passo. (Não reage a
   ninguém → pontuação ruim, de propósito.)
4. Visualiza as trajetórias simuladas como caixas animadas.
5. **Empacota** a simulação nos protos:
   `SimulatedTrajectory` (1 objeto) → `JointScene` (1 rollout) →
   `ScenarioRollouts` (32 rollouts de 1 cenário) e valida com as funções de
   `submission_specs`.
6. **Avaliação local**: calcula as *features* de cada rollout
   (`metric_features`) e depois as métricas finais
   (`metrics.compute_scenario_metrics_for_bundle`).
7. **Submissão**: `SimAgentsChallengeSubmission` por *shard*, arquivos
   `*.binproto-XXXXX-of-YYYYY` (limite de 2 GB do protobuf) e `.tar.gz`.

---

## 4. A técnica de IA

Um sim agent é uma **política generativa condicionada à cena**:
`π(ação_t | histórico, mapa, outros agentes)`, amostrada passo a passo.

| Família | Ideia | Exemplos |
|---|---|---|
| **Behavior cloning** | Treinar a política para imitar o próximo passo dos logs | BC simples, com cuidado com *compounding errors* |
| **Tokenização + modelo autoregressivo** | Discretizar o movimento em "tokens" e treinar um Transformer tipo GPT para prever o próximo token de todos os agentes | MotionLM, Trajeglish, SMART |
| **Difusão** | Gerar trajetórias conjuntas por denoising, com guiagem para evitar colisões | Modelos de difusão de cena |
| **Imitação + RL** | Adicionar recompensas (não colidir, ficar na pista) ao imitation learning | GAIL e variantes, RL fine-tuning |

O desafio central é o **erro composto**: pequenos desvios em cada passo levam a
estados que nunca apareceram no treino (distribution shift), e a política "se
perde". Por isso a malha fechada é muito mais difícil que a predição em malha
aberta.

> 💡 Se você conhece LLMs: um sim agent autoregressivo é um modelo de linguagem
> cujo vocabulário são micro-manobras, gerando "frases" de movimento para todos
> os agentes ao mesmo tempo.

---

## 5. Como se mede sucesso: realismo distribucional

Não existe uma única resposta certa; o que se avalia é se **o futuro real é
provável sob a distribuição dos 32 rollouts**. Para cada *feature*, estima-se um
histograma a partir das simulações e mede-se a verossimilhança do valor real.

| Grupo | Feature | Peso no *metametric* (2025) |
|---|---|---|
| **Cinemática** | velocidade linear | 0,05 |
| | aceleração linear | 0,05 |
| | velocidade angular | 0,05 |
| | aceleração angular | 0,05 |
| **Interação** | distância ao objeto mais próximo | 0,10 |
| | **indicação de colisão** | **0,25** |
| | tempo até colisão (TTC) | 0,10 |
| **Mapa** | distância à borda da pista | 0,05 |
| | **indicação de saída de pista (offroad)** | **0,25** |
| | violação de semáforo | 0,05 |

O **metametric** (score final, entre 0 e 1) é a combinação linear desses scores
(pesos em `challenge_2025_sim_agents_config.textproto`). Note que **colisão e
offroad somam metade do peso** — a métrica é desenhada em torno de segurança.
O ADE também é calculado, mas só como referência (não entra no score).

---

## 6. Armadilhas comuns

- **Remover o primeiro passo** dos estados simulados (ainda é histórico) — o
  formato exige exatamente 80 passos futuros.
- **Todos os agentes simulados devem ser válidos em todos os passos** (não podem
  "sumir").
- Usar informação futura do log dentro da política = vazamento; no teste ela
  não existe.
- Os 32 rollouts precisam ser **diversos**: 32 cópias idênticas geram
  histogramas degenerados e verossimilhança ruim.
- As métricas ficam em `wdl_limited/` (licença própria, restrita ao uso com o
  dataset).

---

## 7. Para ir além

- Substitua a política constante por uma que siga o centro da faixa mais
  próxima com um controlador simples e meça o ganho de offroad/colisão.
- Treine um BC com o modelo do tutorial 10 prevendo só 1 passo à frente e rode-o
  em malha fechada; observe o erro composto.
- Paper: *The Waymo Open Sim Agents Challenge* (Montali et al., NeurIPS 2023
  Datasets & Benchmarks).

---

⬅️ [12 — WOMD multimodal](12-womd-multimodal-lidar-camera.md) · ➡️ [14 — Scenario Gen](14-scenario-gen.md)
