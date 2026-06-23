---
nome: Análise de Causa-Raiz de Degradação
descricao: Cruza configuração, métricas e logs de um serviço para raciocinar até a causa-raiz de uma degradação
versao: 1.0.0
tags: [observabilidade, causa-raiz, incidente, elasticsearch]
inputs:
  - nome: config
    descricao: Arquivo de configuração do cluster/serviço afetado
  - nome: metricas
    descricao: Série temporal de métricas cobrindo a janela do incidente
  - nome: logs
    descricao: Trecho de logs do serviço cobrindo a mesma janela das métricas
---

# Análise de Causa-Raiz de Degradação

## Objetivo
Levar a IA a raciocinar até a **causa-raiz** de uma degradação de serviço — o gatilho real
e o mecanismo que o liga aos sintomas — cruzando três artefatos (configuração, métricas e
logs da mesma janela), em vez de só listar sintomas. O time reusa o prompt a cada
degradação, trocando apenas o pacote de entrada.

## Quando usar
- Numa investigação de incidente/postmortem, quando há config + métricas + logs cobrindo a
  janela do problema e é preciso separar causa de consequência.
- Não é para triagem em tempo real (para isso, ver `nota-de-triagem`); aqui o foco é análise.

## Exemplo de uso / saída (resumo)
Entrada: pacote do Cerebro (cerebro.yaml + 5 pontos de métrica 08:00–10:00 + log do nó).
Saída (Haiku 4.5): RCA estruturada que identifica como causa-raiz o **job de reindexação
travado** (task 88123, ainda em 41% às 09:58, deveria ter terminado ~03:30) competindo por
heap com a indexação de pico → pressão de heap (61%→94%) → GCs longos → circuit breaker a
96% → buscas com timeout e resultado parcial (11/12 shards). Classifica corretamente como
**consequência** (não causa): queda do cache hit (74%→29%), subida do p99 (850→6700ms), GCs
e saturação do thread pool. Ação imediata: cancelar o task de reindex; preventivas: rever
heap, reagendar a janela do reindex e alertar SLA do job.

## Dados sensíveis / sanitização
Decisão aplicada **antes** de enviar ao modelo externo (demonstrada na execução, não só
descrita): foram removidos o **nome do nó interno** (`cerebro-node-3`) e as **classes Java
internas** do Elasticsearch (`o.e.t.LoggingTaskListener`, `o.e.m.j.JvmGcMonitorService`,
etc.), além dos milissegundos dos timestamps (ruído). Foram **preservados** os sinais que
sustentam o diagnóstico: heap, shards, progresso do reindex, métricas e parâmetros de
config. Regra geral: *sanitizar identificadores de infraestrutura e de cliente, preservando
os sinais técnicos que sustentam o diagnóstico.*

## Limitações conhecidas
- Saída aberta: não há resposta única verificável por regex. A qualidade é avaliada por
  **LLM-as-judge** (rubrica do CP09), não por asserts determinísticos.
- A análise é tão boa quanto o pacote de entrada; lacunas nos artefatos viram hipóteses em
  aberto (o prompt pede que elas sejam declaradas).
- Mesmo um output forte pode conter detalhes fabricados — exige revisão humana (ver curadoria).

## Testes (CP09) — gate de qualidade com LLM-as-judge
Saída aberta → avaliada por **juiz LLM**, não por regex. Arquivos: `promptfooconfig.yaml` (gate)
e `calibracao.yaml` + `calibracao/` (evidência da calibração).

- **Rubrica (4 critérios, 0–2, total 0–8):** causa-raiz correta · correlação×causa · ação
  proporcional · honestidade epistêmica. **Gate:** aprova só se **total ≥ 6 E nenhum critério = 0**
  (um zero reprova mesmo com total ≥ 6).
- **Juiz × gerador separados:** juiz = `google:gemini-2.5-flash`; gerador da análise no gate =
  `anthropic:messages:claude-haiku-4-5` (evita autoavaliação).
- **Calibração (contra duas saídas de referência já pontuadas à mão):**
  - `calibracao/rca-boa.md` (humano 2/2/2/2) → juiz **PASS**.
  - `calibracao/rca-ruim.md` (humano 0/0/1/0) → juiz **FAIL** com `C1=0 C2=0 C3=0 C4=0`,
    identificando corretamente o cache tratado como causa (é efeito), o "reiniciar o cluster"
    desproporcional e a certeza fabricada.
  - Diferença máxima juiz × humano: **1 ponto** (no critério de ação) → dentro da tolerância;
    vereditos batem. **Juiz calibrado sem ajuste de nota.**
- **Ajuste feito na calibração (o que quebrou e o conserto):** a 1ª versão da rubrica forçava o
  juiz a "começar com a linha C1=…", em texto livre. O `llm-rubric` do promptfoo espera um JSON
  `{pass, reason}` do grader; o formato livre quebrava o parse e o juiz **reprovava as duas**
  saídas (até a boa). Correção: a rubrica passou a só **descrever os critérios + a regra do
  gate**, deixando o promptfoo cuidar do JSON. Aí os vereditos saíram corretos.
- **Gate em ação:** rodado contra os artefatos do Cerebro (CP03), a análise gerada pelo
  análise gerada (Claude Haiku 4.5) foi avaliada pelo juiz e **aprovada** (PASS) — o gate roda automático a cada execução.

## Curadoria (CP03)
- **Técnica:** Chain-of-Thought **exposto e estruturado**. Diferente de `triagem-de-pods` e
  `nota-de-triagem` (onde o CoT fica interno), aqui o raciocínio é o produto: a saída externa
  a linha do tempo e a cadeia causal.
- **Formato dirigido pelo consumidor:** o consumidor é um engenheiro analisando o incidente,
  não um plantonista agindo em segundos. Ele precisa **ver e verificar** o encadeamento das
  evidências para confiar na causa-raiz — por isso o raciocínio sai visível, em Markdown
  estruturado. Mesmo critério "quem consome?" dos prompts anteriores, com decisão oposta.
- **Escolha de modelo:** **Haiku 4.5 (Anthropic), via Copilot CLI** — modelo pequeno/barato
  e de baixa latência. A aposta era se um modelo desse tier daria conta da tarefa mais pesada
  de raciocínio do playbook; deu (pré-pontuação 8/8 na rubrica do CP09: causa-raiz correta,
  causa×consequência, ação proporcional e honestidade epistêmica). Reforça que um prompt de
  CoT bem construído reduz a dependência de um modelo grande.
- **Reparos encontrados na revisão (mostram por que o gate do CP09 é necessário):**
  1. O modelo justificou aumentar o heap com "benchmarking shows 16GB deixa margem" — não há
     benchmark nos artefatos; é uma justificativa **fabricada** (pequeno desvio de honestidade
     epistêmica). A recomendação é válida, a fundamentação inventada não.
  2. Imprecisões numéricas na seção de limites ("8 horas de atraso" vs. ~6,5h além do prazo;
     "8% de progresso" para uma variação de 38%→41%).
  Conclusão: mesmo um output 8/8 precisa de revisão humana — o que antecipa o valor do juiz
  automático do CP09.
- **Execução:** modelo **Haiku 4.5 (Anthropic)**, rodado sobre o pacote sanitizado do Cerebro.
