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
