---
nome: Triagem de Pods Kubernetes
descricao: Analisa um snapshot de cluster e devolve a triagem dos pods problemáticos com causa provável e próxima ação
versao: 1.0.0
tags: [kubernetes, sre, triagem, observabilidade]
inputs:
  - nome: snapshot_cluster
    descricao: Saída combinada de kubectl get pods + kubectl describe pod + kubectl logs --previous do cluster a ser triado
---

# Triagem de Pods Kubernetes

## Objetivo
Dar ao plantonista de SRE uma triagem rápida e confiável da saúde dos pods de um
namespace. A partir de um snapshot colado (não acessa o cluster), o prompt aponta os
pods problemáticos, infere a **causa provável** cruzando STATUS + eventos + logs (em vez
de só repetir o STATUS) e recomenda a próxima ação, com urgência e confiança.

## Quando usar
- Durante o plantão, ao receber um alerta de pods instáveis e querer um diagnóstico em 1–2 minutos.
- Quem tem acesso ao cluster coleta o snapshot e cola no parâmetro `{{snapshot_cluster}}`.

## Como coletar o snapshot
```
kubectl get pods -n <namespace>
kubectl describe pod <pod-problematico> -n <namespace>
kubectl logs <pod-problematico> -n <namespace> --previous
```
Cole as três saídas concatenadas em `{{snapshot_cluster}}`.

## Exemplo de uso / saída
Entrada (trecho do snapshot — pod reiniciando):
```
sentinel-api-7d9c8b6f4-h4m2t   0/1   CrashLoopBackOff   14 (90s ago)   42m
... Last State: Terminated / Reason: OOMKilled / Exit Code: 137 / Limits: memory 512Mi
... logs: [FATAL] [runtime] out of memory, shutting down process
```
Saída (GPT-5 mini):
```markdown
## Triagem — sentinel-prod  ·  1 pod(s) com problema

### 🔴 sentinel-api-7d9c8b6f4-h4m2t — P1 · confiança alta
- **Status:** CrashLoopBackOff
- **Causa provável:** OOMKilled: processo excede o limite de memória do container (512Mi).
- **Evidências:** describe → 'Terminated/OOMKilled/Exit 137'; logs → '[FATAL] out of memory'
- **Ação:** `kubectl set resources deployment/sentinel-api -n sentinel-prod --limits=memory=1Gi --requests=memory=512Mi && kubectl rollout restart deployment/sentinel-api -n sentinel-prod`
```
Caso saudável (snapshot só com pods Running/READY): a saída é apenas o cabeçalho com a
linha `✅ Nenhum pod problemático.` — inclusive quando há `RESTARTS` antigos em um pod
que está atualmente Running.

## Limitações conhecidas
- Trabalha só com o que está no snapshot; um snapshot incompleto leva a diagnóstico incompleto.
- Não acessa o cluster: os comandos sugeridos precisam ser executados pelo plantonista.
- Casos raros (ex.: problemas intermitentes que não aparecem nos logs `--previous`) podem não ser capturados.

## Testes (CP08)
`promptfooconfig.yaml` nesta pasta, rodado contra os 3 snapshots do CP01 em 2 provedores
(OpenAI gpt-4o-mini + Claude Haiku 4.5). Asserts por entrada: E1 cita `sentinel-api-7d9c8b6f4-h4m2t`
+ causa (OOMKilled/memória); E2 cita os 2 pods + causas (2.9.2/ImagePullBackOff e Insufficient/cpu);
E3 indica que não há pod problemático e não usa o marcador 🔴. Mais latência ≤ 5s e custo ≤ US$ 0,01.
**Resultado: 6 passed / 0 failed / 0 errors.** Setup e ajustes comuns: ver o
[README da categoria](../README.md) (seção *Testes (CP08)*).

## Curadoria (CP01)
- **Técnica:** Chain-of-Thought interno (raciocínio passo a passo sem expor) + saída
  estruturada por template. O raciocínio interno é o que leva à causa provável; o template
  garante saída consistente e legível.
- **Formato:** optou-se por **Markdown** em vez de JSON porque o consumidor desta saída é
  humano (o plantonista lendo no terminal durante o incidente) e porque o Markdown é mais
  econômico em tokens e latência — fator que dialoga diretamente com os limites de custo e
  latência exigidos no CP08. O JSON fica reservado a saídas destinadas a consumo por
  automação. Cabe registrar que esse critério — *formato estruturado para máquina, formato
  legível para humano* — é aplicado **caso a caso, e não como regra global da biblioteca**:
  o prompt `networkpolicy-sentinel` (CP06), por exemplo, deve produzir YAML, pois seu
  consumidor é o próprio Kubernetes. A decisão de formato decorre, portanto, da pergunta
  "quem consome a saída?", e não de preferência estética.
- **Refinamentos até ficar bom:**
  1. A 1ª versão devolvia JSON e produziu **JSON inválido** (aspas duplas não escapadas no
     campo de evidências). Trocamos para Markdown e padronizamos aspas simples nos excertos.
  2. Adicionei a regra explícita "pod Running/READY com restart antigo não é falha" para
     evitar falso positivo no caso saudável.
- **Execução:** modelo **GPT-5 mini**; validado nos 3 snapshots do CP01 (1 pod OOMKilled;
  2 pods ImagePullBackOff + Pending/CPU; e o caso saudável), com resultado correto nos três.
