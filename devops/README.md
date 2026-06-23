# DevOps

Prompts voltados a **infraestrutura, automação e operação** de sistemas: pipelines de CI/CD, containers, orquestração, provisionamento, observabilidade, confiabilidade e segurança operacional.

## Escopo

Entram aqui prompts relacionados a:

- Pipelines de CI/CD (GitHub Actions, GitLab CI, Jenkins etc.).
- Containers e orquestração (Docker, Kubernetes, Helm).
- Infraestrutura como código (Terraform, Pulumi, Ansible).
- Provedores de nuvem (AWS, GCP, Azure) e seus recursos.
- Observabilidade (logs, métricas, tracing, alertas, dashboards).
- Confiabilidade, SRE, postmortems e análise de incidentes.
- Segurança operacional (hardening, secrets, políticas de acesso).

## Fora de escopo

- Escrita de código de aplicação → usar `desenvolvimento/`.
- Conteúdo educacional sobre DevOps (aulas, artigos, vídeos) → usar `criacao-conteudo/`.

## Prompts

| Prompt | Descrição | Versão |
|--------|-----------|--------|
| [triagem-de-pods](triagem-de-pods/) | Triagem de pods Kubernetes a partir de um snapshot | 1.0.0 |
| [nota-de-triagem](nota-de-triagem/) | Nota padronizada de alerta a partir de um alerta cru | 1.0.0 |
| [causa-raiz](causa-raiz/) | Análise de causa-raiz cruzando config, métricas e logs | 1.0.0 |
| [estrategia-de-backpressure](estrategia-de-backpressure/) | Comparação de estratégias de backpressure no barramento | 1.0.0 |
| [migracao-batch-para-streaming](migracao-batch-para-streaming/) | Cadeia de migração de pipeline batch → streaming | 1.0.0 |
| [networkpolicy-sentinel](networkpolicy-sentinel/) | Endurecimento de NetworkPolicy permissiva (default-deny) | 1.0.0 |

## Testes (CP08) — setup e ajustes comuns

Os prompts de saída estruturada têm um `promptfooconfig.yaml` ao lado do `prompt.md`
(o teste viaja junto com o prompt). Rodados com `npx promptfoo@0.121.17 eval --no-cache`.

- **Provedores do gate determinístico (dois fornecedores distintos):**
  `openai:gpt-4o-mini` + `anthropic:messages:claude-haiku-4-5`. Durante o desenvolvimento passei
  por Gemini free tier e Groq, mas consolidei o gate em OpenAI + Anthropic por confiabilidade
  (ver ajuste 6). O **juiz** (CP09) permanece em `gemini-2.5-flash`.
- **Limites operacionais em todos:** latência ≤ 5s e custo ≤ US$ 0,01.

**Ajustes feitos durante o CP08 (o caminho, não só o destino):**
1. **Ollama (local) → Groq.** Comecei com Llama local (Ollama) pela privacidade e custo zero;
   na máquina disponível a inferência ficou lenta/instável demais para o orçamento de 5s.
   Troquei por Groq (mesma família Meta/Llama, na nuvem) — o trade-off privacidade-local ×
   confiabilidade-nuvem na prática.
2. **`gemini-2.0-flash` → `gemini-2.5-flash`.** O 2.0-flash retornava `limit: 0` no free tier
   (HTTP 429 → backoff de 60s, que parecia "travamento"). O 2.5-flash é permitido no tier.
3. **`thinkingBudget: 0` no Gemini.** O 2.5-flash faz *reasoning* por padrão e estourava os 5s
   de latência (≈4,5k tokens de reasoning). Desliguei o thinking — modelo permitido **e** rápido.
   É o trade-off custo/latência × modelo que o CP08 manda pesar.
4. **Assert de custo tolerante.** O `type: cost` nativo do promptfoo dá erro em provedores sem
   tabela de preço (Groq, e modelo local). Reescrevi como `javascript` que valida ≤ US$ 0,01
   quando há custo (Gemini) e passa quando o provedor não reporta.
5. **Erro 503 transitório** (Google "high demand") apareceu numa chamada da `nota-de-triagem` —
   não é regressão de prompt. Fica como nota para o gate do CP10: erro transitório de infra
   (503/429) ≠ regressão; o pipeline não deve reprovar o build por isso.
6. **Consolidação de provedores no gate de CI (CP10).** No pipeline, o Gemini free tier
   intermitentemente estourava a latência (>5s) e dava 529/503 por throttling/overload —
   derrubando o build por motivo **ambiental**, não regressão (o conteúdo gerado estava correto);
   o Groq free, por sua vez, não reporta custo. Consolidei o **gate determinístico** em
   `openai:gpt-4o-mini` + `anthropic:messages:claude-haiku-4-5` (ambos rápidos, confiáveis e com
   billing por API), mantendo `-j 1` (serial). O juiz informativo segue no Gemini — calibrado no
   CP09, e como não barra o build, um throttle nele não causa falso vermelho.

**Resultados (`promptfoo eval`):**

| Config | Casos × provedores | Resultado |
|--------|--------------------|-----------|
| triagem-de-pods | 3 × 2 | **6 passed / 0 failed** |
| networkpolicy-sentinel | 1 × 2 | **2 passed / 0 failed** |
| nota-de-triagem | 3 × 2 | **5 passed / 0 failed / 1 erro 503 transitório** |
