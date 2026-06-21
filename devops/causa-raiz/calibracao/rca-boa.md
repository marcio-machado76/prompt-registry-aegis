# Análise de causa-raiz — Cerebro / degradação de busca

## Resumo
O job de reindexação agendado para 02:00 não terminou no SLA esperado (~90 min) e continuou rodando durante o horário comercial, competindo por heap com a indexação de alta demanda. Isso provocou pressão progressiva de memória (61% → 94%), GCs cada vez mais agressivas, saturação do thread pool de escrita e, finalmente, disparo do circuit breaker e falhas de busca a partir de 09:58 UTC.

## Linha do tempo e correlação
- 02:00 — reindex job [88123] inicia (schedule da config).
- 08:02 — reindex em 38% (3.8M/10M); heap 61%, p99 850ms → já deveria ter terminado ~03:30.
- 08:14 — GC young 620ms (pressão inicial de heap).
- 08:41 — throttling de indexing no shard logs-2026.05[7].
- 09:03 — write thread pool 150/200; indexed_docs 9800/s.
- 09:12 — GC old 1.1s; heap 79%.
- 09:31 — heap 86% (HierarchyCircuitBreaker "approaching limit").
- 09:58 — write pool cheio 200/200, rejeitando bulk; heap 94%, p99 6700ms; circuit breaker disparou (7.7gb/8gb, 96%); search retornou resultado parcial (11/12 shards); reindex ainda em 41%.
- 10:01 — CircuitBreakingException (7.9gb/8gb). 10:05 — all shards failed em 3 de 20 queries.

## Causa-raiz
**Gatilho:** o job de reindexação [88123] (02:00) não completou seu SLA de 90 min e seguia em apenas 41% (4.1M/10M) às 09:58 — ~8h de runtime, muito além do prazo de ~03:30.
**Mecanismo até os sintomas:**
1. Reindex ativo + indexing paralelo (4200→12400 docs/s) competem pelo heap de 8GB.
2. Heap comprimido (61%→94%) → GCs cada vez mais longas (620ms→1.8s).
3. GC longo + thread pool de escrita saturado (150→200/200) → rejeição de bulks (EsRejectedExecutionException).
4. Heap a 96% → circuit breaker dispara → buscas com timeout (5031ms > 5000ms) e resultado parcial (11/12 shards).
5. Pressão de heap → eviction agressiva do query cache → hit ratio 74%→29%.

## Causa vs. consequência
- **CAUSA-RAIZ:** reindexação não concluída + pressão de indexação saturando o heap.
- **EFEITOS (não a causa):** queda do cache hit (74→29) — consequência da eviction; subida do p99 (850→6700ms) — consequência das pausas de GC e contenção; GCs longas — consequência da pressão de heap; fila de escrita 200/200 — consequência dos GCs travando workers. O cache baixo NÃO causou a lentidão; ambos são efeitos da saturação de heap.

## Ação recomendada
**Imediata:** cancelar/pausar o reindex task para liberar heap — `POST /_tasks/<task_id>/_cancel`; aguardar o heap descomprimir e confirmar p99 e cache voltando.
**Preventiva:** reagendar a reindexação para janela sem indexing de pico; rever o heap/limites (8GB pode ser insuficiente para 12 shards + reindex + indexing concorrente); adicionar alerta de SLA do job (reindex que ultrapassa ~150 min dispara aviso).

## Limites da análise
Os dados não permitem concluir **por que** a reindexação ficou tão lenta (throttle de I/O, CPU, pausa/retomada manual não aparecem nos logs); a causa do pico de indexação (4200→12400 docs/s) não está clara; não se sabe se outros índices além de logs-2026.05 foram afetados; e a plataforma/infra exata (versão, recursos) não é informada. São hipóteses em aberto, não fatos.
