# Catálogo de prompts

Coleção de prompts em Markdown organizados por categoria/área de domínio. Cada prompt vive em sua própria pasta, contendo o arquivo `prompt.md` (texto puro, pronto para copiar e colar) e um `README.md` com metadados, variáveis e exemplos de uso.

Este repositório faz parte do material dos projetos da pós-graduação em AIOps e Inteligência Artificial com Engenharia Cloud: [pos.veronez.io/pos-aiops](https://pos.veronez.io/pos-aiops/).

Convenções de estrutura, nomenclatura e manutenção estão em [`CLAUDE.md`](./CLAUDE.md).

## Playbook de IA operacional (Aegis)

Este repositório é um playbook de IA para engenharia, montado **a partir do template
[prompt-registry](https://github.com/fabricioveronez/prompt-registry)** (usado como ponto de
partida e referência de convenções). Os seis prompts dos Checkpoints 01–06 foram mapeados
nas convenções do template assim:

- **Categoria → pasta na raiz:** todos os prompts são operacionais, então caem em
  [`devops/`](./devops/) — sem aninhar categorias.
- **Um prompt por pasta, nomeada pelo resultado:** `triagem-de-pods/`, `causa-raiz/`,
  `networkpolicy-sentinel/` etc. — nunca pela técnica (não existe `chain-of-thought/`).
- **`{{placeholders}}` = `inputs` do frontmatter:** os parâmetros de cada prompt (o snapshot,
  o alerta, os artefatos, o cenário) aparecem como `{{variavel}}` no `prompt.md` e estão
  documentados no campo `inputs`. É onde "todo prompt é parametrizável" encontra a estrutura.
- **`versao: 1.0.0` em todo prompt:** versão inicial; daqui pra frente toda mudança passa por
  commit semântico com escopo na categoria (ex.: `feat(devops): adiciona prompt de triagem de pods`).
- **Desvio consciente — cadeia:** `migracao-batch-para-streaming/` é uma cadeia de prompts
  (CP05), então usa uma subpasta `prompts/` com os elos numerados, em vez de um único
  `prompt.md`. O frontmatter da cadeia fica no `README.md` da pasta. Está registrado lá.
- **Testes viajam junto com o prompt:** a partir do CP08, cada prompt de saída estruturada
  ganha um `promptfooconfig.yaml` ao lado do `prompt.md`; o gate de qualidade (CP09) e o
  pipeline (CP10) completam a virada de "texto solto" para "ativo testado do time".

## Como usar

1. Navegar até a categoria de interesse.
2. Abrir o `README.md` do prompt para entender objetivo, variáveis esperadas e limitações.
3. Copiar o conteúdo do `prompt.md` e substituir os placeholders `{{nome_variavel}}` pelos valores desejados.

## Adicionando um prompt

Use o slash command [`/catalogar`](./.claude/commands/catalogar.md) passando o texto do prompt como argumento. Ele analisa, propõe organização (categoria, slug, frontmatter) e, após sua aprovação, escreve os arquivos e atualiza os índices — sem commitar. Convenções completas em [`CLAUDE.md`](./CLAUDE.md).

## Categorias

### [Desenvolvimento](./desenvolvimento/)

Escrita, revisão e refatoração de código, design de APIs e arquitetura, debugging, testes e documentação técnica.

_Nenhum prompt cadastrado ainda._

### [DevOps](./devops/)

Pipelines de CI/CD, containers, orquestração, infraestrutura como código, observabilidade, SRE e segurança operacional.

- [triagem-de-pods](./devops/triagem-de-pods/) — triagem de pods Kubernetes a partir de um snapshot do cluster.
- [nota-de-triagem](./devops/nota-de-triagem/) — nota padronizada de alerta a partir de um alerta cru.
- [causa-raiz](./devops/causa-raiz/) — análise de causa-raiz cruzando config, métricas e logs.
- [estrategia-de-backpressure](./devops/estrategia-de-backpressure/) — comparação de estratégias de backpressure num barramento.
- [migracao-batch-para-streaming](./devops/migracao-batch-para-streaming/) — cadeia de prompts para migrar um pipeline batch → streaming.
- [networkpolicy-sentinel](./devops/networkpolicy-sentinel/) — endurecimento de uma NetworkPolicy permissiva (default-deny).

### [Produtividade](./produtividade/)

Organização pessoal, gestão de tempo e tarefas, rotina, hábitos, foco e decisões sobre fluxo de trabalho individual.

_Nenhum prompt cadastrado ainda._

### [Finanças](./financas/)

Orçamento, investimentos, planejamento financeiro, impostos e apoio a decisões financeiras.

_Nenhum prompt cadastrado ainda._

### [Criação de Conteúdo](./criacao-conteudo/)

Roteiros, artigos, posts para redes sociais, material didático e copy de divulgação.

_Nenhum prompt cadastrado ainda._

<!--
Ao adicionar um prompt, substituir "Nenhum prompt cadastrado ainda" pela lista:

- [nome-do-prompt](./<slug-da-categoria>/<slug-do-prompt>/) — o que o prompt faz, em uma linha.
-->

## Contribuindo

Antes de adicionar ou alterar um prompt, revisar [`CLAUDE.md`](./CLAUDE.md) — a seção **Manutenção da documentação** lista todos os arquivos que precisam ser atualizados junto com a mudança (este índice incluso).
