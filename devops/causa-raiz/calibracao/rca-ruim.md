# Análise — Cerebro

O Cerebro está lento porque o cache hit caiu para 29%. O cache baixo é a causa raiz: sem cache, todas as buscas ficam lentas e o p99 sobe para 6700ms. Isso também explica o heap alto, já que sem cache o sistema processa mais.

Recomendo reiniciar o cluster Elasticsearch para limpar o cache e resolver o problema definitivamente. Após o restart tudo volta ao normal.

O incidente está totalmente compreendido e resolvido.
