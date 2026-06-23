---
nome: Endurecimento de NetworkPolicy
descricao: Recebe uma NetworkPolicy permissiva e a reescreve em default-deny com regras mínimas comentadas
versao: 1.0.0
tags: [kubernetes, seguranca, networkpolicy, hardening]
inputs:
  - nome: manifesto
    descricao: Manifesto de NetworkPolicy permissivo a ser endurecido
  - nome: regras
    descricao: Regras do padrão de segurança que a versão corrigida precisa respeitar
  - nome: mapa_servicos
    descricao: Mapa de como cada serviço é identificado no cluster (namespace, labels, portas)
---

# Endurecimento de NetworkPolicy

## Objetivo
Receber uma NetworkPolicy permissiva (allow-all disfarçado) e devolvê-la endurecida:
default-deny explícito + apenas os fluxos mínimos necessários, com os seletores corretos e
um comentário por regra. Artefato crítico de segurança, por isso passa por verificação e
refino iterativo antes de virar item de playbook.

## Quando usar
- Ao revisar/corrigir um manifesto de NetworkPolicy que está permissivo demais.
- O engenheiro cola o manifesto, as regras do padrão e o mapa de serviços nos parâmetros e
  recebe a versão endurecida — depois conduz o ciclo de verificação (ver abaixo).

## Exemplo de uso / saída
Entrada (`{{manifesto}}` permissivo): `podSelector: {}` com `ingress: [{}]` e `egress: [{}]`
no namespace `sentinel-prod` (libera tudo). Mais `{{regras}}` (padrão de segurança) e
`{{mapa_servicos}}` (namespaces, labels e portas de Relay, API gateway, Forge, Cerebro, DNS).

Saída final (v2, Sonnet 4.6) — trecho:
```yaml
# default-deny explícito: bloqueia todo ingress/egress de todos os pods de sentinel-prod
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: sentinel-prod }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# (allow-egress) Permite saída para o Forge (forge-prod, app=forge, 5432 TCP) — warehouse
- to:
    - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: forge-prod } }
      podSelector: { matchLabels: { app: forge } }
  ports: [{ protocol: TCP, port: 5432 }]
```
A política final tem 3 documentos (default-deny + allow-ingress + allow-egress), com
`namespaceSelector` por `kubernetes.io/metadata.name`, DNS em UDP+TCP/53 e comentário em
toda regra. (YAML completo registrado na entrega do desafio.)

## Iterações de verificação e refino (CP06)
O valor deste item está no ciclo dirigido — não na primeira saída.

- **v1 (geração):** já saiu sólida. Sonnet 4.6 acertou pontos delicados de cara: usou
  `kubernetes.io/metadata.name` (selecionar namespace por label, não por nome), combinou
  `namespaceSelector` + `podSelector` no mesmo item `from`/`to` (AND, não OR), e liberou DNS
  em UDP **e** TCP/53. Default-deny separado, comentário em toda regra, zero `- {}`.
- **Verificação (a IA revisou a própria saída como revisor de segurança):**
  - ❌ **Ingress sem restrição de porta** — Relay e API gateway alcançam qualquer porta do
    Sentinel. A porta de escuta do Sentinel não está no mapa → não inventar, **sinalizar**.
  - ⚠️ **`kubernetes.io/metadata.name` exige cluster ≥ 1.21** — em clusters anteriores a label
    não existe e o seletor falha silenciosamente, bloqueando tráfego legítimo. (Achado afiado.)
  - ⚠️ **DNS TCP/53** — legítimo (respostas grandes/DNSSEC), mas merece revisão consciente.
  - ❌ **FALSO POSITIVO — "comentários ausentes":** a revisão apontou ausência de comentários,
    mas isso foi artefato da entrada: o YAML colado no prompt de crítica vinha sem os `#` (a v1
    real os tinha em toda regra). Registrado de propósito: **o humano precisa verificar até a
    crítica** — a IA se auto-revisar não basta, alguém confere se a revisão bate com a realidade.
- **v2 (refino):** adicionou o comentário `# ATENÇÃO` sobre a porta ausente nos dois ingress,
  o aviso de cluster ≥ 1.21 no topo (com fallback para label customizada), justificativa do
  DNS TCP/53 (RFC 7766), e **preservou** todos os comentários (em vez de "adicioná-los" como se
  faltassem). Os seletores e portas corretos foram mantidos sem alteração.

## Limitações conhecidas
- Endurece com base no que está em `{{regras}}` e `{{mapa_servicos}}`; dado faltante (ex.: a
  porta de escuta do Sentinel) vira comentário de advertência, não uma regra completa.
- Pressupõe cluster Kubernetes ≥ 1.21 para o seletor de namespace por `kubernetes.io/metadata.name`.

## Testes (CP08)
`promptfooconfig.yaml` nesta pasta, rodado contra o manifesto permissivo + regras + mapa do CP06
em 2 provedores (OpenAI gpt-4o-mini + Claude Haiku 4.5). Asserts na NetworkPolicy gerada:
`kind: NetworkPolicy` com `Ingress` e `Egress`, sem allow-all (`- {}`), egress libera Forge (5432)
e Cerebro (9200), ingress libera `app: relay`, toda regra com comentário (`#`) e custo ≤ US$ 0,01 (latência informativa). **Resultado: 2 passed / 0 failed / 0 errors.** Setup e ajustes comuns: ver o
[README da categoria](../README.md) (seção *Testes (CP08)*).

## Curadoria (CP06)
- **Técnica:** geração + **self-critique / refino iterativo** dirigido por humano. O prompt da
  biblioteca produz a v1; o ciclo de verificação (a IA critica a própria saída, levantando as
  perguntas de um revisor de segurança) e o humano curando os achados produzem a v2.
- **Escolha de modelo:** **Claude Sonnet 4.6 (Anthropic)** — raciocínio forte para acertar
  seletores e semântica de NetworkPolicy, custo médio; em segurança, qualidade pesa mais que
  economia. Quarto provedor distinto na matriz do playbook não foi necessário — reusei Anthropic.
- **Formato dirigido pelo consumidor:** saída em **YAML**, porque o consumidor é o **Kubernetes**
  (máquina). É o contraponto explícito do `triagem-de-pods`, onde o consumidor era humano e a
  saída foi Markdown — mesma pergunta ("quem consome?"), resposta oposta.
- **Lição de método registrada:** a revisão automática trouxe um achado afiado (cluster ≥ 1.21)
  *e* um falso positivo (comentários). Isso confirma por que o gate do CP09 precisa de calibração
  humana: a IA revisora é útil, mas não infalível — a curadoria humana sobre a crítica é parte do
  processo, não um extra.
