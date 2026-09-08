# dados.gov.pt

Pipeline de ingestão do catálogo nacional de dados abertos, a partir da API pública do
[dados.gov.pt](https://dados.gov.pt). É o pipeline mais recente do repositório e o **exemplar de
referência da metodologia**: runner metadata-driven, `%%configure` em todos os notebooks, escrita
sempre por path ABFSS, e um star schema completo na Gold.

Quem for construir um pipeline novo deve ler esta página a par de
[Visão Geral da Metodologia](../metodologia/visao-geral.md), e usar estes notebooks como modelo.

---

## O percurso completo

```
API pública dados.gov.pt                    Ficheiros de referência
  /datasets/  /harvest/sources/  /reuses/     CAOP (.xlsx) · 3 GeoJSON de concelhos
        │                                             │
        │  GET paginado, JSON preservado              │  cópia byte a byte
        ▼                                             ▼
BRONZE  (Lakehouse de ingestão)
  Tables/dados_gov/dados_gov_bronze_datasets
  Tables/dados_gov/dados_gov_bronze_harvest_sources     Files/dados_gov/ingest/caop/
  Tables/dados_gov/dados_gov_bronze_reuses              Files/dados_gov/ingest/geojson/
        │  cast por schema_map · limpeza de NULL · origem + dt_ingest
        ▼
SILVER  (mesmo Lakehouse do Bronze)
  dados_gov_silver_datasets        dados_gov_silver_geografia    ← lê o .xlsx da CAOP
  dados_gov_silver_harvest_sources dados_gov_silver_centroides   ← lê os GeoJSON + a geografia
  dados_gov_silver_reuses
        │  13 steps, um por tabela do modelo
        ▼
GOLD    (Lakehouse e workspace diferentes)
  6 dimensões · 3 factos · 4 pontes
        │
        ▼
  Refresh do modelo semântico Power BI
```

Bronze e Silver partilham o mesmo Lakehouse. A Gold vive noutro Lakehouse, **noutro workspace** —
e é alcançada exactamente da mesma maneira, por um path ABFSS vindo do `main_set`. É por isso que
a separação não custa nada.

---

## Configuração

Um ficheiro, `dados_gov.json`, em `Files/configs/pipelines/` do Lakehouse de configuração. O
runner recebe apenas o nome (`"config_file": "dados_gov"`) e vai lá buscá-lo
([ver Orquestração](../metodologia/orquestracao-runner.md#onde-vive-o-ficheiro)).

| Bloco | Conteúdo |
|---|---|
| Raiz | `workspace: e_ext_ingest_ws`, `pipeline_id: dados_gov`, `log_Level: {logLevel: 5, printOnScreen: true}` |
| `allTasks` | `lh_bronze`, `lh_silver`, `lh_gold`, `folder_destino: Files/dados_gov/ingest` |
| `groups` | Três, por camada |

| Grupo | `run_withouth_new_data` | Tasks |
|---|---|---|
| `dados_gov-bronze` | `true` | 1 — `dados_gov_BRONZE` |
| `dados_gov-silver` | `false` | 5 — uma por notebook Silver |
| `dados_gov-gold` | `false` | 2 — `dados_gov_GOLD` e um `sm_refresh` |

O primeiro grupo tem de ter `run_withouth_new_data: true`, senão nada corre: o runner arranca com
`new_data = 0`.

---

## Bronze

Task única, `movement_type: external_to_bronze`. Tudo corre dentro do notebook — a fonte é pública
e não precisa de gateway ([Tipo 2](../metodologia/camada-bronze.md#tipo-2--rest-api-pública-directa)).

**Parâmetros de rede, todos no `datasource` da task, nenhum no código:**

| Parâmetro | Valor |
|---|---|
| `page_size` | 100 |
| `timeout` | 120 |
| `max_retries` | 5 |
| `backoff_seconds` | 5 |
| `sleep_seconds` | 0.3 |
| `detail_workers` | 5 |

A paginação lê o `total` da primeira resposta e calcula quantas páginas faltam; as restantes são
descarregadas em paralelo com um `ThreadPoolExecutor` de `detail_workers`, sobre uma sessão
`requests` reutilizada. Um 5xx é repetido com espera crescente até `max_retries`, e ao fim disso
falha com o URL e o número de tentativas na mensagem.

**Aterragem.** As três respostas JSON são escritas como tabelas Delta preservando a estrutura
aninhada — nada é achatado. Só se acrescentam duas colunas técnicas, `_ingested_at` e
`_source_endpoint`. Os ficheiros de referência (CAOP e GeoJSON) são copiados tal como estão para
`Files/`, sem serem lidos.

**Uma decisão que vale a pena perceber.** O script original que deu origem a este pipeline
enriquecia as métricas de cada dataset com valores lidos de um segundo endpoint. Isso não foi
replicado no Bronze: substituir um campo pelo valor de outra chamada é corrigir conteúdo, e o
Bronze é cópia fiel. Os parâmetros desse enriquecimento ficaram na configuração, disponíveis para
a Silver os usar sobre dados já aterrados.

**Resultado devolvido:**

```json
{ "status": "OK", "errorId": 0,
  "datasets_loaded": <n>, "sources_loaded": <n>, "reuses_loaded": <n>,
  "caop_copiado": true, "geojson_copiados": <n>,
  "new_data": true }
```

O `new_data` é calculado como "alguma das três extracções trouxe registos". É esta chave, e só
esta, que decide se a Silver e a Gold correm — o runner não olha para as contagens
([ver Camada Bronze](../metodologia/camada-bronze.md#resultado-devolvido-ao-runner)).

---

## Silver

Cinco notebooks, um por tabela, todos com `movement_type: bronze_to_silver` e um `schema_table`
próprio na task. Três lêem tabelas Delta do Bronze; dois lêem ficheiros.

| Notebook | Lê | Escreve |
|---|---|---|
| `dados_gov_silver_datasets` | `dados_gov_bronze_datasets` | `dados_gov_silver_datasets` |
| `dados_gov_silver_harvest_sources` | `dados_gov_bronze_harvest_sources` | `dados_gov_silver_harvest_sources` |
| `dados_gov_silver_reuses` | `dados_gov_bronze_reuses` | `dados_gov_silver_reuses` |
| `dados_gov_silver_geografia` | o `.xlsx` da CAOP, em `Files/` | `dados_gov_silver_geografia` |
| `dados_gov_silver_centroides` | os GeoJSON + `dados_gov_silver_geografia` | `dados_gov_silver_centroides` |

**Ficheiros que o Spark não lê directamente.** A CAOP é um `.xlsx` e os concelhos são GeoJSON.
Nenhum se lê por um path `abfss://` com as bibliotecas habituais. O padrão usado é ler os bytes com
o Spark e entregá-los à biblioteca em memória:

```python
raw = spark.read.format("binaryFile").load(path).select("content").head()["content"]
```

Ver [Fontes de Dados](../metodologia/fontes-de-dados.md#ficheiros-que-não-se-lêem-por-abfss).

**Normalização.** Cada notebook declara um `schema_map` de `{input, output, type}` e uma função
genérica aplica-o. Como o Bronze preservou o JSON aninhado, o `input` aceita notação com ponto
(`organization.id`). Campos que não existam geram a coluna a `null` com aviso, em vez de rebentar.
Estruturas que se querem inteiras — arrays de tags, listas de recursos — passam com `type: "raw"`,
sem cast, para a Gold as explodir.

**Auditoria.** Todas as tabelas levam `origem` e `dt_ingest`. Não levam `ficheiro_origem` nem
`data_ficheiro`: a fonte é uma tabela de nome fixo, não um ficheiro datado, e essas colunas só
fazem sentido quando há uma data no nome do ficheiro
([ver Camada Silver](../metodologia/camada-silver.md#colunas-de-auditoria)).

**Escrita.** `overwrite` com `overwriteSchema`, por `.save(path)`. A Silver é reconstruída de raiz
a cada execução — não guarda histórico.

---

## Gold

Um notebook, `dados_gov_GOLD`, com **13 steps** — um por tabela do modelo. A lista está na
configuração, em `running_steps`, e cada bloco começa por verificar se o seu nome lá está.

| Tipo | Tabelas | Origem |
|---|---|---|
| Dimensões | `dim_calendario` | gerada, não lê Silver |
| | `dim_geografia`, `dim_centroides` | as Silver homónimas |
| | `dim_organizacao` | `silver_datasets` + `silver_reuses` |
| | `dim_harvester` | `silver_harvest_sources` |
| | `dim_tag` | `silver_datasets` + `silver_reuses` |
| Factos | `fact_datasets`, `fact_resources` | `silver_datasets` |
| | `fact_reuses` | `silver_reuses` |
| Pontes | `ponte_dataset_geografia`, `ponte_dataset_tag` | `silver_datasets` |
| | `ponte_reuse_dataset`, `ponte_reuse_tag` | `silver_reuses` |

Todas as tabelas do modelo vivem no mesmo notebook, cada uma escrita por `.save(path)` com
`overwrite`. É o padrão descrito em
[Camada Gold → Star Schema num Único Notebook](../metodologia/camada-gold.md#star-schema-num-único-notebook):
as tabelas partilham leituras e funções de harmonização, e a ordem entre elas é a das células.

O `fact_resources` existe porque o grão é outro. A Silver guardou o array `resources` de cada
dataset sem lhe tocar; é a Gold que o explode, porque uma linha por recurso não é uma linha por
dataset. Regra geral: mudar de grão é trabalho da Gold.

**Resultado devolvido:**

```json
{ "status": "OK", "results": [ { "step": "<nome do step>", "rows": <linhas escritas> } ] }
```

Uma entrada por step executado, pela ordem em que correram.

O `results` é acumulado célula a célula e passado ao `handleError` em cada `except`, para que uma
falha a meio devolva na mesma o registo dos steps que já tinham corrido.

Depois da Gold, uma task de `type: sm_refresh` dispara o refresh do modelo semântico Power BI e
espera pela conclusão.

---

## Como correr

**O pipeline inteiro:** abrir `dados_gov_runner` e correr. É o único ponto de entrada.

**Uma camada só:** no runner, usar `local_updates` para desligar os grupos que não interessam, sem
tocar no ficheiro de configuração:

```python
local_updates = [
    {"group_order": 1, "task_order": None, "key": "enabled", "new_value": False},
]
```

**Uma tabela Gold só:** abrir o `dados_gov_GOLD` directamente, pôr `running_local = True` e deixar
em `running_steps`, na célula de parâmetros, apenas os steps que se quer reconstruir.

**Um notebook Silver só:** abrir o notebook, `running_local = True`, e correr. Os valores por
defeito da célula `parameters` são os de desenvolvimento.

Em execução manual, `running_local = True` faz com que um erro levante excepção com o traceback
completo, em vez de terminar o notebook em silêncio
([ver Orquestração](../metodologia/orquestracao-runner.md#execução-local--correr-um-notebook-isolado)).

---

## O que este pipeline demonstra da metodologia

| Regra | Onde se vê |
|---|---|
| [Runner metadata-driven](../metodologia/orquestracao-runner.md#as-duas-gerações-de-runner) | `nb_settings` + `read_config_pipelines`, sem configuração no notebook |
| [Escrita por path, nunca pelo catálogo](../metodologia/principios.md#escrita-por-path-nunca-pelo-catálogo) | Zero `saveAsTable` em qualquer camada |
| [`%%configure` com o Lakehouse de suporte](../metodologia/orquestracao-runner.md#a-célula-configure) | Célula 0 de todos os notebooks, incluindo o runner |
| [Anatomia do notebook de camada](../metodologia/orquestracao-runner.md#anatomia-de-um-notebook-de-camada) | Mesma ordem de células nos oito notebooks |
| [Tudo parametrizável](../metodologia/visao-geral.md#tudo-parametrizável--sem-valores-hardcoded) | Até os parâmetros de rede da API estão na configuração |
| [Bronze sem transformação](../metodologia/camada-bronze.md#responsabilidade) | Enriquecimento de métricas deliberadamente adiado para a Silver |

---

[Home](../Home.md)
