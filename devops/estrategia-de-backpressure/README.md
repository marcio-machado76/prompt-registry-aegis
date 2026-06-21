---
nome: Estratégia de Backpressure
descricao: Recebe o estado e as restrições de um barramento e compara estratégias de backpressure antes de recomendar
versao: 1.0.0
tags: [arquitetura, filas, backpressure, decisao]
inputs:
  - nome: cenario
    descricao: Estado atual do barramento (throughput, pico e duração, retenção, consumidores) mais as restrições do time
---

# Estratégia de Backpressure

## Objetivo
Apoiar uma decisão de arquitetura: como conter a sobrecarga de um barramento de eventos
quando um produtor envia mais do que os consumidores processam. Em vez de cuspir uma
resposta única, o prompt força a IA a **comparar** pelo menos três estratégias (prós,
contras, custo e aderência a cada restrição) antes de recomendar — o raciocínio vale tanto
quanto a recomendação.

## Quando usar
- Diante de incidentes de sobrecarga/lag num barramento, quando há mais de um caminho
  defensável e a decisão é cara.
- O engenheiro cola o estado + as restrições em `{{cenario}}` e recebe um documento de decisão.

## Exemplo de uso / saída (resumo)
Entrada: cenário do Relay (180k/s sustentado, pico 320k/s por 25min, retenção 4h,
consumidores Forge e Sentinel; SLA Sentinel 60s, Forge tolera 15min, orçamento +8%,
não-perda inegociável).
Saída (GPT-5.3 codex): documento de decisão comparando 5 estratégias (priorização,
backpressure na origem, spillover/DLQ, particionamento por tenant, autoscaling), descartando
shed/drop por violar a não-perda, e recomendando uma combinação em camadas — priorizar o
Sentinel (SLA 60s), backpressure na origem, spillover para o Forge (dentro dos 15min) e
autoscaling seletivo — ancorada na folga real de bufferização (25min de pico ≪ 4h de retenção).

## Dados sensíveis / sanitização
Avaliação registrada: sensibilidade **baixa**. O cenário traz codinomes de arquitetura
(Relay, Forge, Sentinel) — abstratos, não são hostnames, IPs, credenciais nem dados de
cliente — e métricas de capacidade agregadas. Não há identificador de tenant/cliente nem
segredo. Decisão: nada crítico a redigir; os codinomes poderiam ser pseudonimizados se a
topologia de arquitetura for tratada como confidencial, mas as métricas de capacidade são
**preservadas** porque são a substância da análise. Contraponto consciente ao `causa-raiz`
(CP03), onde havia um identificador concreto de infraestrutura a remover.

## Limitações conhecidas
- Saída aberta: não há resposta única verificável por regex. A qualidade é avaliada por
  **LLM-as-judge** (camada do CP09, estendida no CP10), não por asserts determinísticos.
- A recomendação depende de dados que o cenário pode não trazer (ex.: throughput de consumo
  efetivo, custo unitário de escala); o prompt é instruído a declarar essa incerteza.

## Curadoria (CP04)
- **Técnica:** análise comparativa de trade-offs / documento de decisão. O prompt obriga a
  enumerar ≥3 estratégias, avaliar cada uma contra cada restrição e só então recomendar.
- **Formato dirigido pelo consumidor:** o consumidor é um tomador de decisão técnico (CTO).
  Por isso o formato é um documento de decisão com BLUF (recomendação resumida no topo) e a
  análise que a sustenta abaixo. Mesmo critério "quem consome?" dos demais prompts.
- **Escolha de modelo:** **GPT-5.3 codex (OpenAI), via Copilot CLI.** Codex é afinado para
  código/agentes; aqui a tarefa é decisão de arquitetura (domínio técnico adjacente), então a
  escolha é defensável, mas exigiu validar se a comparação saiu equilibrada (e não enviesada
  para "implementar X"). Segundo provedor já garantido pelos CPs anteriores.
- **Refino v1 → v2 (durante a criação):** a 1ª geração do prompt **removeu os exemplos de
  estratégias** que estavam no meta-prompt; resultado: o modelo omitiu o particionamento por
  tenant e a DLQ e preencheu a lista com um strawman (shed/drop) e não conectou o pico de
  25min à retenção de 4h. Refinei o prompt: (1) instrução para avaliar a folga
  pico×retenção; (2) re-semeei o espaço de estratégias (incluindo DLQ e particionamento por
  tenant); (3) regra anti-strawman (estratégia que viola restrição inegociável é descartada
  de saída, sem ocupar slot). O v2 endereçou os três pontos.
- **Observação de qualidade:** a recomendação final empilha quatro mecanismos — é defesa em
  camadas (cada um cobre uma restrição distinta: priorização→SLA, backpressure→origem,
  spillover→não-perda, autoscaling→folga), e não redundância; vale ter o olho para não virar
  "recomendar tudo".
- **Execução:** modelo **GPT-5.3 codex (OpenAI)**, rodado sobre o cenário do Relay; v2 após
  uma rodada de refino.
