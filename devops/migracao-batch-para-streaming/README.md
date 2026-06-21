---
nome: Migração de Batch para Streaming (cadeia)
descricao: Cadeia de prompts que migra um pipeline de processamento em lote para orientado a eventos, em passos reversíveis
versao: 1.0.0
tags: [arquitetura, streaming, migracao, prompt-chaining]
inputs:
  - nome: estado_atual
    descricao: Descrição do pipeline em lote hoje (job, etapas, destino, dependências, pontos frágeis) e o que a migração precisa garantir
---

# Migração de Batch para Streaming (cadeia)

Este prompt é uma **cadeia** (prompt chaining): cada elo recebe a saída do anterior
como entrada. Os elos vivem em [`prompts/`](./prompts):

1. [`01-diagnostico.md`](./prompts/01-diagnostico.md) — diagnostica o estado atual. Entrada: `{{estado_atual}}`.
2. [`02-plano-migracao.md`](./prompts/02-plano-migracao.md) — propõe o passo a passo. Entrada: saída do elo 1.
3. [`03-plano-executavel.md`](./prompts/03-plano-executavel.md) — detalha o plano executável e reversível. Entrada: saída do elo 2.

> Desvio consciente da convenção "um prompt.md por pasta": como o CP05 é uma cadeia,
> os elos ficam em `prompts/` numerados (mesmo padrão do exemplo `service-topology-mapper`
> em `prompt-registry-extended`). O frontmatter da cadeia fica neste README.

## Objetivo
Migrar um pipeline de processamento em lote para orientado a eventos (streaming) sem
big-bang, em passos reversíveis. Como a tarefa é grande demais para um prompt só, ela é
quebrada numa cadeia: cada elo resolve uma parte e entrega a próxima.

## Quando usar
- Ao planejar a migração de um pipeline batch para streaming, quando é preciso manter os
  consumidores funcionando durante a transição e poder voltar atrás a cada etapa.
- O engenheiro roda os três elos em sequência, colando a saída de um na entrada do próximo.

## Exemplo de uso / saída (resumo)
Entrada do Elo 1 (`{{estado_atual}}`): o cenário do Forge (job cron de 60min, 14 etapas
Spark, tabelas particionadas por hora; dependentes Sentinel, Cerebro e billing da Pepper;
ponto frágil: lote que falha dobra o volume do próximo).

Execução (Haiku 4.5), encadeada:
1. **Diagnóstico** → estado atual + 3 dependentes + ponto frágil (cascata de volume) +
   requisitos (não-perda, reversibilidade, reprocessamento dentro da retenção de 4h do Relay)
   + suposições declaradas (SLA do Sentinel, plataforma de streaming indefinida).
2. **Plano em passos** → 6 passos incrementais em modo dual: preparar infra → ingestão dual
   → transformações em paralelo → switch do Sentinel (crítico, primeiro) → switch de
   Cerebro/Pepper → desligar o batch por último. Cada passo com modo de compatibilidade
   (feature flag + fallback para batch) e rollback.
3. **Runbook executável** → por passo: ações concretas, validação observável (lag, paridade
   de dados/alertas, latência) e rollback mecânico (`yarn application -kill`, flag = false,
   `TRUNCATE`), mais critério de transição entre passos.

(Saídas completas de cada elo registradas na entrega do desafio.)

## Limitações conhecidas
<!-- TODO -->

## Testes
Prompt de **saída aberta**: não há resposta única verificável por regex, então não tem
`promptfooconfig.yaml` determinístico (o CP08 cobre só os 3 prompts de saída estruturada).
A avaliação é por **LLM-as-judge** — a camada montada no CP09 e estendida a todos os
prompts no CP10.

## Curadoria (CP05)
- **Técnica:** prompt chaining (encadeamento de prompts). A migração é grande demais para um
  prompt único — jogada de uma vez, a resposta sai rasa. Quebrá-la em diagnóstico → plano →
  runbook deixa cada elo focado e usa a saída do anterior como entrada, ganhando profundidade.
- **Por que três elos nesta ordem:** separa *entender* (elo 1), *decidir a estratégia* (elo 2)
  e *detalhar a execução* (elo 3). O elo 2 é proibido de descer a comando/código (isso é do
  elo 3), o que mantém cada saída no nível certo de abstração.
- **Custo na cadeia:** instruí cada elo a produzir saída **enxuta e estruturada** (tópicos,
  não prosa) e a consumir só a saída do anterior — sem re-injetar o cenário cru nos três elos.
  Numa cadeia, saída inchada de um elo vira entrada cara do próximo; o custo compõe.
- **Escolha de modelo:** **Haiku 4.5 (Anthropic), via Copilot CLI** — tier barato/rápido, já
  validado no CP03. Tarefa de planejamento não exige frontier model, e a cadeia faz 3 chamadas,
  então custo por chamada importa. As três rodaram por fração de centavo.
- **Coerência observada entre elos:** o diagnóstico amarrou o reprocessamento à retenção de
  **4h do Relay** (o mesmo número do CP04) sem ser instruído; o plano e o runbook carregaram
  esse fato adiante. A cadeia preservou contexto de ponta a ponta.
- **Reparos na revisão humana (o valor agregado depois da cadeia):**
  1. Os **números de exemplo** do runbook são ilustrativos e internamente inconsistentes
     (usa "108M rows" para 10 min e para 1 hora). São placeholders marcados como "resultado
     esperado" — trocar pelos números do ambiente real.
  2. O runbook **assume uma stack concreta** (Kafka, yarn, LaunchDarkly, Redshift/Snowflake)
     que o diagnóstico tinha sinalizado como desconhecida; está declarado como suposição nos
     pré-requisitos, mas quem executar deve substituir pelos comandos da stack real.
- **Decisão de formato:** os três elos saem em **Markdown estruturado** porque o consumidor é
  um time de engenharia executando a migração — o elo 3, em especial, é um runbook que precisa
  ser lido e seguido sob pressão.
