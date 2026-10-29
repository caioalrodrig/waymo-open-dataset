# 14 — Scenario Gen: gerar cenários de tráfego do zero

| | |
|---|---|
| **Notebook** | [`tutorial_scenario_gen.ipynb`](../../tutorial/tutorial_scenario_gen.ipynb) |
| **Dataset** | Motion (WOMD), formato `Scenario` (proto) — split de validação |
| **Nível** | 4 · ★★★★★ |
| **Pré-requisitos** | [13 — Sim Agents](13-sim-agents.md) (mesma infraestrutura de submissão e métricas) |
| **Código-chave** | `utils/sim_agents/submission_specs.py` (`ChallengeType.SCENARIO_GEN`), `wdl_limited/sim_agents_metrics/challenge_2025_scenario_gen_config.textproto`, `protos/sim_agents_submission.proto` |
| **Desafio original** | Scenario Gen (2025) |

---

## 1. O problema no mundo real

Sim Agents parte de uma cena real e simula como ela continua. Mas a frota só
grava as situações que de fato aconteceram — e as situações **perigosas e raras**
são justamente as que menos aparecem nos logs.

**Scenario generation** dá o passo seguinte: dado apenas um trecho de mapa,
o estado dos semáforos e o que o AV fez, **inventar o resto do trânsito** —
quantos agentes, onde começam, de que tamanho, e como se movem por 9 s. Usos:

- **Ampliar a cobertura de testes** com cenas plausíveis que nunca foram
  gravadas;
- **Criar variações** de uma cena (mais pedestres, um ciclista a mais);
- No futuro, **gerar cenários adversariais** sob controle (ex.: um carro furando
  o sinal) para estressar o planejador.

---

## 2. A tarefa

| Item | Valor |
|---|---|
| **Entradas permitidas** | Mapa (roadgraph), histórico de semáforos (passado e atual), histórico do AV (passado e atual), **número de agentes por tipo** (veículo, pedestre, ciclista) |
| **Proibido usar** | O histórico dos outros agentes, que existe nos arquivos de validação/teste — o notebook fornece utilitários para **remover essa informação privilegiada** |
| **Saída por agente** | Tipo, dimensões (`length, width, height`) e trajetória completa `x, y, z, heading` |
| Horizonte | **91 passos a 10 Hz** (passado + atual + futuro: o cenário inteiro) |
| Rollouts | **32 gerações** por cenário |

Comparação com Sim Agents:

| | Sim Agents (13) | Scenario Gen (14) |
|---|---|---|
| Agentes iniciais | Dados (do log) | **Gerados** |
| Estados iniciais | Dados | **Gerados** |
| Tamanho/tipo | Dados | **Gerados** (só a contagem por tipo é dada) |
| Passos gerados | 80 (só futuro) | 91 (cena inteira) |
| Métricas | Realismo distribucional | Mesmas features e pesos |

---

## 3. O que o notebook faz, passo a passo

1. Carrega um cenário da validação e plota seus tracks.
2. Lê a configuração de submissão de *scenario gen* (passos, frequência,
   rollouts).
3. **Prepara a entrada legítima**: conta agentes por tipo
   (`num_agents_per_type`) e cria uma cópia do cenário **sem os dados
   privilegiados**, restaurando apenas o histórico do AV.
4. **Baseline aleatório**: para cada agente, sorteia posição e velocidade
   iniciais e simula velocidade constante. (Não respeita o mapa nem reage →
   pontuação ruim, de propósito.)
5. Visualiza os cenários gerados.
6. Empacota em `ScenarioRollouts` (agora com tipo e dimensões nos
   `SimulatedTrajectory`) e valida.
7. Avalia localmente com as mesmas features/métricas do Sim Agents e mostra o
   `metametric`.
8. Gera a submissão por shard (`.binproto`) e o `.tar.gz`.

---

## 4. A técnica de IA

É um problema de **modelagem generativa estruturada**: gerar um conjunto de
tamanho variável de objetos com atributos contínuos e trajetórias, condicionado
a um mapa e a um agente (o AV).

| Abordagem | Ideia |
|---|---|
| **Dois estágios** | (1) gerar estados iniciais (posições/tamanhos) com um modelo de densidade sobre o mapa; (2) rodar uma política de sim agents a partir deles |
| **Autoregressivo sobre agentes** | Inserir um agente por vez, condicionado aos já inseridos (como gerar objetos numa imagem um a um) |
| **Difusão de cena** | Gerar todos os agentes e trajetórias juntos por denoising, condicionando no mapa/AV |
| **Modelos de tokens** | Tokenizar mapa, agentes e movimentos e treinar um Transformer generativo único |

> 💡 Analogia com CV: é como passar de "completar um vídeo a partir dos primeiros
> frames" (Sim Agents) para "gerar um vídeo inteiro a partir de um layout de
> cena" (Scenario Gen).

---

## 5. Como se mede sucesso

Mesmo arcabouço de **realismo distribucional** do [13](13-sim-agents.md): features
cinemáticas, de interação e de mapa, comparadas com a distribuição real, com os
mesmos pesos (colisão e offroad = 0,25 cada) combinados no **metametric**
(`challenge_2025_scenario_gen_config.textproto`). Os agentes de
`tracks_to_predict` são avaliados, mas todos os agentes gerados participam das
features de interação.

---

## 6. Armadilhas comuns

- **Vazamento**: usar o histórico real dos outros agentes (posições iniciais)
  invalida o resultado. Use os utilitários que limpam o cenário.
- O número de agentes por tipo **deve bater** com o informado.
- Agora as dimensões e o tipo fazem parte da saída — esquecê-los gera submissões
  inválidas.
- Gerar pedestres no meio da rodovia ou carros dentro de prédios destrói a
  métrica de offroad: use o mapa desde o primeiro estágio.

---

## 7. Para ir além

- Baseline melhor: amostre posições iniciais de veículos ao longo de centros de
  faixa (`map_features.lane`) com espaçamento realista e heading alinhado à faixa;
  pedestres sobre calçadas/faixas de pedestre.
- Combine com o seu sim agent do tutorial 13 para a fase de movimento.

---

⬅️ [13 — Sim Agents](13-sim-agents.md) · ➡️ [15 — Direção end-to-end](15-e2e-driving.md)
