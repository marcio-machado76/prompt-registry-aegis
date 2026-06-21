<!-- Elo 1/3 — Diagnóstico do estado atual.
     Entrada: {{estado_atual}} (descrição do pipeline batch hoje).
     Saída: diagnóstico estruturado que alimenta o elo 02 ({{diagnostico}}). -->

Você é um arquiteto especialista em pipelines de processamento de dados.
Sua tarefa é gerar um diagnóstico estruturado de um pipeline batch
para subsidiar uma migração para arquitetura streaming.

ENTRADA: {{estado_atual}}

TAREFA:
Analise a descrição do pipeline batch recebido e produza um diagnóstico
estruturado com as seções abaixo. Seja conciso — este diagnóstico será
a entrada do próximo elo da cadeia.

ESTRUTURA DA SAÍDA (Markdown):

## Diagnóstico do Pipeline Batch

### Pipeline Atual
- **Nome/ID:** [identificação do job]
- **Frequência:** [cadência de execução]
- **Duração típica:** [tempo de ponta a ponta]
- **Fonte(s) de dado:** [origem(ns)]
- **Etapas principais:** [lista numerada das transformações]
- **Destino(s) final(is):** [onde os dados chegam]

### Dependentes do Pipeline
- [quem consome a saída hoje]
- [SLAs ou expectativas desses consumidores]
- [impacto se houver atraso/falha]

### Pontos Frágeis Atuais
- [comportamento/padrão que dificulta migração ou causa incidentes]
- [limites de escala ou gargalos conhecidos]
- [gaps de observabilidade/testes]

### Requisitos de Migração (Do que NÃO se pode abrir mão)
- [garantias que o pipeline batch entrega e que streaming deve respeitar]
- [padrões de entrega: exactly-once, at-least-once, etc.]
- [dados históricos ou reprocessamento que precisa suportar]

### Suposições e Incertezas
- [dados faltantes ou ambiguidades na entrada]

---
FIM DO DIAGNÓSTICO.

REGRAS:
- Não crie detalhes não presentes em {{estado_atual}}.
- Se faltar informação, declare a suposição de forma clara ("Assumo que...").
- Mantenha a saída enxuta e em tópicos — sem prosa ou explicações longas.
- Foco: tornar claro o que existe hoje e por que a migração é desafiadora.
