# Playbook de IA operacional — Aegis

Biblioteca de prompts de engenharia da Aegis, **versionada, testada e tratada como código**:
cada prompt é parametrizável (recebe os dados por parâmetro), documentado e coberto por testes
automatizados, para que qualquer pessoa do time pegue e confie no resultado — e o ativo
sobreviva à saída de quem o escreveu.

> Montado **a partir do template [prompt-registry](https://github.com/fabricioveronez/prompt-registry)**
> (usado como ponto de partida e referência de convenções; a linhagem está preservada no
> histórico de commits). Daqui em diante o repositório fala como o playbook da Aegis.

## Origem e mapeamento (Checkpoints 01–06)

Os seis prompts nasceram dos checkpoints do desafio e foram mapeados nas convenções do template:

- **Categoria → pasta na raiz:** todos os prompts são operacionais, então caem em
  [`devops/`](./devops/) — sem aninhar categorias.
- **Um prompt por pasta, nomeada pelo resultado:** `triagem-de-pods/`, `causa-raiz/`,
  `networkpolicy-sentinel/` etc. — nunca pela técnica (não existe `chain-of-thought/`).
- **`{{placeholders}}` = `inputs` do frontmatter:** os parâmetros de cada prompt (o snapshot,
  o alerta, os artefatos, o cenário) aparecem como `{{variavel}}` no `prompt.md` e estão
  documentados no campo `inputs`. É onde "todo prompt é parametrizável" encontra a estrutura.
- **Desvio consciente — cadeia:** `migracao-batch-para-streaming/` é uma cadeia de prompts
  (CP05), então usa uma subpasta `prompts/` com os elos numerados, em vez de um único
  `prompt.md`. O frontmatter da cadeia fica no `README.md` da pasta.

## Estrutura e convenções

```
<categoria>/
  <nome-do-prompt>/
    prompt.md            # frontmatter + texto puro do prompt (placeholders {{variavel}})
    README.md            # mesmo frontmatter + doc humana (objetivo, uso, exemplo, limitações)
    promptfooconfig.yaml # testes do prompt (onde aplicável) — viajam junto com o prompt
```

- **Categoria** = pasta na raiz, em `kebab-case`, uma por domínio, sem aninhar.
- **Prompt** = subpasta nomeada pelo **resultado**, não pela técnica.
- **`prompt.md`** = frontmatter YAML + o texto do prompt, com os parâmetros como `{{variavel}}`.
- **`README.md`** = o mesmo frontmatter + documentação humana (objetivo, quando usar, exemplo,
  limitações, curadoria).
- **Frontmatter** (idêntico nos dois arquivos): `nome`, `descricao`, `versao` (semver, inicia
  em `1.0.0`), `tags`, `inputs` (lista de parâmetros, cada um com `nome` e `descricao`).
- **Versionamento:** o campo `versao` de cada prompt + **commits semânticos** com escopo na
  categoria (ex.: `feat(devops): adiciona prompt de triagem de pods`). Os índices (este README
  e o da categoria) são atualizados junto com a mudança.

## Categorias

### [DevOps](./devops/)

Pipelines de CI/CD, containers, orquestração, infraestrutura como código, observabilidade,
SRE e segurança operacional.

- [triagem-de-pods](./devops/triagem-de-pods/) — triagem de pods Kubernetes a partir de um snapshot do cluster.
- [nota-de-triagem](./devops/nota-de-triagem/) — nota padronizada de alerta a partir de um alerta cru.
- [causa-raiz](./devops/causa-raiz/) — análise de causa-raiz cruzando config, métricas e logs.
- [estrategia-de-backpressure](./devops/estrategia-de-backpressure/) — comparação de estratégias de backpressure num barramento.
- [migracao-batch-para-streaming](./devops/migracao-batch-para-streaming/) — cadeia de prompts para migrar um pipeline batch → streaming.
- [networkpolicy-sentinel](./devops/networkpolicy-sentinel/) — endurecimento de uma NetworkPolicy permissiva (default-deny).

> Este playbook é só de DevOps; as demais categorias do template foram removidas. A convenção
> "uma categoria por domínio" segue valendo — uma nova pasta nasce quando houver prompt de outro domínio.

## Testes e integração contínua

O playbook é testado, não só guardado:

- **Determinístico (CP08):** `triagem-de-pods`, `nota-de-triagem` e `networkpolicy-sentinel`
  têm `promptfooconfig.yaml` com asserts de formato + limites de latência e custo.
- **LLM-as-judge (CP09):** `causa-raiz` (saída aberta) tem um gate de qualidade por juiz LLM,
  calibrado contra pontuação humana (ver `devops/causa-raiz/`).
- **Pipeline (CP10):** [`.github/workflows/promptfoo.yml`](./.github/workflows/promptfoo.yml)
  roda a suíte a cada pull request — os asserts determinísticos barram o build, o juiz LLM é
  informativo. A justificativa do desenho do gate está em
  [`.github/workflows/JUSTIFICATIVA.md`](./.github/workflows/JUSTIFICATIVA.md).

## Como usar

1. Abrir o `README.md` do prompt para entender objetivo, parâmetros e limitações.
2. Copiar o conteúdo do `prompt.md` e substituir os placeholders `{{variavel}}` pelos valores
   reais (o snapshot, o alerta, os artefatos…).
3. Rodar no modelo de sua escolha (chat, playground ou API).

## Como adicionar ou alterar um prompt

1. Escolher a categoria existente que melhor se encaixa (ou criar uma nova pasta na raiz, em
   `kebab-case`, com seu próprio `README.md` de escopo).
2. Criar a pasta do prompt nomeada pelo **resultado** e escrever `prompt.md` + `README.md` com
   o **mesmo frontmatter** (`versao: 1.0.0` no nascimento).
3. Adicionar `promptfooconfig.yaml` quando o prompt for testável.
4. Atualizar os índices (este README e o da categoria) e commitar com mensagem semântica
   (`feat(<categoria>): ...`). Evoluções posteriores incrementam o campo `versao`.
