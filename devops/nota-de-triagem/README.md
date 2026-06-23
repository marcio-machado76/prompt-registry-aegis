---
nome: Nota de Triagem de Alerta
descricao: Transforma um alerta cru do Sentinel em uma nota de triagem padronizada de cinco campos
versao: 1.0.0
tags: [observabilidade, alerting, plantao, padronizacao]
inputs:
  - nome: alerta_cru
    descricao: Texto cru do alerta disparado (linha de log/evento) a ser convertido em nota padronizada
---

# Nota de Triagem de Alerta

## Objetivo
Padronizar a nota que o plantonista escreve a cada alerta. A partir de um alerta cru, o
prompt produz uma nota de cinco campos fixos (ALERTA / IMPACTO / HIPÓTESE INICIAL /
AÇÃO IMEDIATA / ESCALAR PARA), para que quem assume o turno seguinte sempre leia no mesmo
formato.

## Quando usar
- A cada alerta disparado pelo sistema de observabilidade, antes de registrar a triagem.
- O plantonista cola a linha crua do alerta em `{{alerta_cru}}` e recebe a nota pronta.

## Exemplo de uso / saída
Entrada (`alerta_cru`):
```
2026-05-13 03:11:00 UTC [Relay] ingest reject rate 6% for 8min, tenant wakanda-systems,
buffer saturated after deploy 02:55
```
Saída (Gemini):
```
ALERTA: Relay - taxa de rejeição de ingestão em 6% por 8min no tenant wakanda-systems
IMPACTO: telemetria degradada e potencial perda de dados para o tenant wakanda-systems
HIPÓTESE INICIAL: deploy realizado às 02:55 causou a saturação do buffer de ingestão
AÇÃO IMEDIATA: rollback do deploy das 02:55 iniciado
ESCALAR PARA: @relay-core se a rejeição não cair em 10min
```

## Limitações conhecidas
- A nota é tão boa quanto o alerta cru: um alerta sem contexto (tenant, horário, deploy)
  gera HIPÓTESE INICIAL mais fraca.
- A HIPÓTESE INICIAL é uma inferência, não um diagnóstico confirmado — serve para orientar
  a primeira ação, não para fechar causa-raiz (isso é o prompt `causa-raiz`).

## Testes (CP08)
`promptfooconfig.yaml` nesta pasta, rodado contra os 3 alertas crus do CP02 em 2 provedores
(OpenAI gpt-4o-mini + Claude Haiku 4.5). Asserts: contém os 5 rótulos (ALERTA / IMPACTO / HIPÓTESE
INICIAL / AÇÃO IMEDIATA / ESCALAR PARA), regex `ESCALAR PARA:.*@\w+`, ≤ 8 linhas, latência ≤ 5s,
custo ≤ US$ 0,01. **Resultado: 5 passed / 0 failed** (+1 erro 503 transitório do Google numa
chamada — instabilidade de servidor, não regressão de prompt). Setup e ajustes comuns: ver o
[README da categoria](../README.md) (seção *Testes (CP08)*).

## Curadoria (CP02)
- **Técnica:** few-shot (híbrido). Os três exemplos do padrão da Aegis ficam embutidos no
  prompt como demonstração de formato/tom, e uma descrição curta trava os cinco rótulos na
  ordem e a regra do `@handle`. Os exemplos são fixos (são o padrão do time), então não são
  parâmetro — o único parâmetro é `{{alerta_cru}}`.
- **Por que few-shot e não só descrição:** formato, tom técnico e concisão se ensinam melhor
  por imitação do que por instrução. A descrição sozinha deixaria o estilo a cargo do modelo.
- **O few-shot ensina mais que o formato — ensina o nível de inferência:** os três exemplos
  do padrão já são inferências, não transcrições do alerta ("deploy reduziu o buffer", "pico
  do tenant saturou o consumer"). Ao embuti-los, demonstra-se o raciocínio esperado: ler o
  alerta, inferir a causa provável e propor a ação proporcional. O modelo replicou esse
  comportamento — na execução do Alerta 1, propôs "aumentar o limite de réplicas" sem que o
  alerta cru dissesse isso. Essa inferência é segura porque cai sob o rótulo HIPÓTESE INICIAL
  (hipótese, não fato) e porque a AÇÃO IMEDIATA é sempre a primeira medida reversível ou
  investigativa — o que limita o custo de uma inferência eventualmente errada.
- **Formato:** texto puro, idêntico ao padrão consolidado pelo time — não Markdown nem JSON.
  O consumidor é humano (o próximo plantonista) e a nota circula por várias superfícies de
  incidente (chat, paginador, ticket, terminal); texto puro é o mais portável e evita que
  marcações apareçam literais onde não há render. É o mesmo critério "quem consome a saída?"
  do prompt `triagem-de-pods`, aqui resultando em texto puro porque o formato já é prescrito.
- **Refinamentos:**
  1. Removi "modelo de IA" da role (`Você é um especialista em SRE...`) para reforçar a
     persona e reduzir hedging.
  2. Travei explicitamente "sem Markdown, sem JSON, apenas as linhas da nota" para os asserts
     de `contains` do CP08.
- **Validação observada:** o Alerta 2 é quase idêntico ao 1º exemplo few-shot; confirmou-se
  que o modelo **adapta** (tenant wakanda-systems, 6%/8min) em vez de copiar o exemplo.
- **Execução:** modelo **Gemini (Google), via Copilot CLI**; validado nos 3 alertas crus do
  CP02, com os 5 rótulos, `@handle` válido e ≤ 8 linhas em todos.
