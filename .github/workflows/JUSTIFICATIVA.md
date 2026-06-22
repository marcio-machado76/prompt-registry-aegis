# CP10 — Justificativa do desenho do gate

O pipeline ([`promptfoo.yml`](./promptfoo.yml)) roda a suíte promptfoo a cada PR e barra
regressão. As decisões de gate são abertas — abaixo, cada uma com ao menos duas alternativas,
o que se ganha e o que se perde, e por que a escolhida.

## 1. O que faz o build falhar?

| Alternativa | Ganha | Perde |
|---|---|---|
| **(escolhida) Só asserts determinísticos barram; juiz LLM é informativo** | Gate reproduzível e confiável; um PR bom nunca é reprovado por flutuação do juiz | Uma regressão de qualidade que *só* o juiz pegaria não bloqueia o merge sozinha |
| Juiz também barra (com threshold) | Pega regressão de qualidade automaticamente | Juiz é não-determinístico — pode reprovar um build por azar (falso vermelho), corroendo a confiança no CI |

**Por quê:** o gate precisa ser determinístico para ser confiável. O juiz (CP09) flutua entre
execuções; usá-lo como bloqueio transforma ruído em build vermelho. Então ele **roda e aparece
no comentário do PR** (sinal de qualidade visível ao revisor), mas quem **barra** é o assert
determinístico. A regressão de qualidade fica coberta pela combinação *comentário do juiz +
revisão humana*. Se um dia quisermos o juiz barrando, o caminho é dar **margem** (ex.: corte em
5/8 em vez de 6/8) e **retentativas** para absorver a flutuação — registrado, mas não adotado agora.

## 2. Suíte inteira ou só os prompts alterados?

| Alternativa | Ganha | Perde |
|---|---|---|
| **(escolhida) Suíte inteira a cada PR** | Simples; pega regressão indireta (ex.: mudança de convenção que afeta vários prompts); suíte pequena = centavos e segundos | Desperdício quando a biblioteca crescer muito |
| Só prompts alterados (path filters) | Rápido e barato em escala | Complexidade de detectar arquivos mudados; **deixa passar** regressão de algo compartilhado que não foi "alterado" diretamente |

**Por quê:** hoje a suíte é pequena (poucas chamadas, modelos free/baratos), então rodar tudo
custa quase nada e elimina o risco de regressão indireta passar. O "só o alterado" vira útil
quando o repositório tiver dezenas de prompts e o tempo/custo de CI pesar — aí o trade-off
inverte (economia vs risco de cobertura).

## 3. Gatilhos, custo e chaves

| Decisão | Escolha | Alternativas consideradas |
|---|---|---|
| **Gatilho** | `pull_request` + `workflow_dispatch` | Adicionar `push: main` (re-roda após merge — redundante se o PR já barrou, mas pega push direto); `schedule` noturno (pega *drift* de provedor) — descartados por ora para não gastar à toa |
| **Chaves** | GitHub **Secrets** do repo (`GEMINI_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`) | Hardcode no YAML (**nunca** — vazamento); OIDC/cofre externo (exagero para chaves de free tier) |
| **Custo por PR** | Modelos free/baratos (Gemini free tier, Groq free) → ~centavos; rodar só em PR (não em todo push) já corta execuções | Rodar em todo push dobraria o custo sem ganho real (o PR é onde a regressão é barrada antes do merge) |

O `workflow_dispatch` foi adicionado para rodar a suíte **manualmente** (re-checar após um erro
transitório, validar sem abrir PR) — custo zero, conveniência alta.

## 4. Action oficial do promptfoo × workflow explícito

| Alternativa | Ganha | Perde |
|---|---|---|
| `promptfoo/promptfoo-action` oficial | Menos YAML; comenta no PR automaticamente | É single-config e **falha o build em qualquer assert, inclusive o do juiz** → incompatível com a decisão a1 |
| **(escolhido) Workflow explícito com `npx promptfoo`** | Controle total: separa o passo que **barra** (determinístico) do passo **informativo** (juiz `continue-on-error`); roda múltiplos configs | Mais YAML para manter |

**Por quê:** a decisão a1 (juiz não barra) **exige** separar gating de informativo — o que a
action oficial, no modelo "falha em qualquer assert", não permite. Por isso o workflow explícito.
O comentário no PR é feito via `actions/github-script` (sem depender da action).

## 5. Erros transitórios de provedor (503/429)

Risco real, já observado no CP08: uma chamada pode falhar por **alta demanda (503)** ou **rate
limit (429)** — não é regressão do prompt, mas derrubaria o gate determinístico (falso vermelho).

| Mitigação | Decisão |
|---|---|
| Retentativas embutidas do promptfoo (backoff em 429/5xx) | **Adotada** (baseline — o promptfoo já reexecuta) |
| Re-rodar via `workflow_dispatch` quando um flake derrubar o build | **Adotada** (botão manual já existe) |
| Envolver o job num "retry-action" agressivo | **Descartada** — re-tentar demais mascara falha real e gasta tokens |

A distinção *regressão × flake transitório* é parte do desenho: o gate confia nas retentativas
do promptfoo e, no limite, no re-run manual — sem retry agressivo que esconda regressão de verdade.

**Caso real observado:** num run, o assert de **latência** do Gemini reprovou (>5s) com o
**conteúdo correto** — era throttling do free tier (concorrência 4 disparava chamadas em rajada,
estourava o RPM → backoff → latência inflada). Não é regressão de prompt. Correção adotada:
rodar o eval **serial (`-j 1`)** no pipeline, espaçando as chamadas para não estourar o RPM —
assim a latência medida reflete a resposta real do modelo, não o tempo de espera por throttling.
Alternativa se ainda flutuar (descartada por ora): tornar latência/custo **informativos** no CI
(só o conteúdo barra), já que latência depende da carga do provedor, não da qualidade do prompt.

## Cobertura de testes da biblioteca

| Prompt | Tipo de teste | No gate? |
|---|---|---|
| triagem-de-pods | determinístico (CP08) | **barra** |
| nota-de-triagem | determinístico (CP08) | **barra** |
| networkpolicy-sentinel | determinístico (CP08) | **barra** |
| causa-raiz | LLM-as-judge (CP09) | informativo |
| estrategia-de-backpressure | LLM-as-judge (padrão CP09), `promptfooconfig.yaml` próprio | informativo |
| migracao-batch-para-streaming | LLM-as-judge no elo 1 (diagnóstico) da cadeia, `promptfooconfig.yaml` próprio | informativo |

Os seis prompts têm `promptfooconfig.yaml`. Os três de saída aberta usam juiz LLM (padrão do
CP09) e, por serem informativos (a1), rodam no passo informativo do pipeline sem barrar o
build — entram como sinal de qualidade no PR. Para a cadeia de migração, o teste avalia o
**elo 1 (diagnóstico)** como representativo da entrada da cadeia.
