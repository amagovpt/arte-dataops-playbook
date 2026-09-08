# Princípios Arquitecturais

Princípios estratégicos que guiam todas as decisões de implementação nos pipelines Fabric. Qualquer novo pipeline deve ser avaliado contra estes critérios.

---

## Thin Pipelines, Fat Notebooks

Ferramentas proprietárias de plataforma (Fabric Pipelines, Data Factory Gen2, Dataflow Gen2) representam vendor lock-in e devem ser usadas **apenas como scheduler e trigger básico** — "apenas chama o Notebook."

Toda a lógica de transformação, orquestração de sub-tarefas e validação de dados vive dentro de notebooks Python/PySpark:

- O runner é um notebook que invoca outros notebooks via `notebookutils.notebook.run()` — não um pipeline visual
- As transformações são código Python explícito, não actividades visuais de "Lookup", "Filter", "ForEach"
- O código é testável localmente (`running_local = True`) sem dependência da plataforma

**Status:** implementado, com uma excepção — quando a fonte só é alcançável via gateway (PBIGW), a chamada à API/BD tem de sair do notebook para uma activity do Fabric Pipeline (`WebActivity`) ou para um Fabric CopyJob, porque o notebook não consegue atravessar essa ligação directamente. Ver [Camada Bronze — Padrões de Ingestão](camada-bronze.md#padrões-de-ingestão), [Orquestração Runner](orquestracao-runner.md) e [Visão Geral — Sem Vendor Lock-in](visao-geral.md#sem-vendor-lock-in--notebooks-em-vez-de-data-factory--dataflow-gen2).

---

## Minimizar Dependência de Componentes Proprietários (Low-Code)

Evitar assets proprietários específicos de plataforma. Quando utilizados, providenciar forma de rapidamente serem portados para lógica de código aberto.

Privilegiar:
- Notebooks Python/PySpark
- SQL standard
- Bibliotecas Python open source (`requests`, `json`, `re`, etc.)

**Status:** implementado — sem Data Factory Gen2, sem Dataflow Gen2, sem conectores visuais.

---

## Abstracção da Camada de Dados

### Formato aberto e portável

Utilizar **Delta Lake** como formato de armazenamento — standard de mercado aberto, compatível com Databricks, Azure Synapse, e outros engines além do Fabric.

**Status:** implementado — todas as tabelas Silver e Gold estão em formato Delta.

### Caminhos relativos e parametrizados

Utilizar caminhos abstractos e configuráveis para facilitar a troca de storage no futuro. Nenhum path hardcoded nos notebooks — todos chegam via `main_set`:

```python
# ✅ Parametrizado — troca de Lakehouse sem tocar no código
lh_bronze = main_set["lh_bronze"]

# ❌ Hardcoded — impossível portar
lh_bronze = "abfss://33f78fb2-abc@onelake.dfs.fabric.microsoft.com/<lakehouse_guid>"
```

**Status:** implementado, com uma excepção conhecida — o path do ficheiro de configuração está
escrito no código da `read_config_pipelines`, com o workspace e o Lakehouse fixos. Ver
[Orquestração → Onde vive o ficheiro](orquestracao-runner.md#onde-vive-o-ficheiro) e
[Decisões Arquitecturais](visao-geral.md#tudo-parametrizável--sem-valores-hardcoded).

### Escrita por path, nunca pelo catálogo

**Regra transversal a todas as camadas (Bronze, Silver, Gold e dims):** as tabelas Delta são sempre escritas por path ABFSS, nunca registadas por nome no catálogo.

O path de uma tabela tem sempre a mesma forma, seja em que camada for: o Lakehouse vem do
`main_set`, o resto é literal.

```python
# ✅ Path ABFSS derivado do main_set — dados e metadados no mesmo Lakehouse
path = f"{lh_gold}/Tables/<schema>/<tabela>"
df.write.format("delta").mode("overwrite").option("overwriteSchema", "true").save(path)

# ❌ saveAsTable — registo no catálogo do Lakehouse por defeito
df.write.format("delta").mode("overwrite").saveAsTable("schema.tabela")

# ❌ Igualmente proibido — a combinação das duas coisas
df.write.format("delta").option("path", path).saveAsTable("tabela")

# ❌ CREATE TABLE, com ou sem LOCATION — registo explícito no catálogo
spark.sql(f"""CREATE TABLE tabela USING DELTA LOCATION '{path}'""")
```

O `.save(path)` cria a tabela Delta física no path indicado: os ficheiros Parquet e o
`_delta_log`. Estes metadados Delta não são os metadados do catálogo. `saveAsTable`,
`writeTo` e `CREATE TABLE` registam adicionalmente um nome no catálogo — mesmo quando
`CREATE TABLE` recebe um `LOCATION` — e não fazem parte do padrão deste projecto.

**Porquê:** quando o registo no catálogo usa o Lakehouse **por defeito** (o declarado no
`%%configure`) e os dados são escritos no path de destino, os metadados do catálogo ficam num
Lakehouse e a tabela Delta física noutro. A segunda execução falha com
`DELTA_TABLE_LOCATION_MISMATCH`.

Para a forma do path ABFSS — incluindo a regra de que um GUID nunca leva o sufixo
`.Lakehouse` — ver [Visão Geral → Sempre ABFSS](visao-geral.md#sempre-abfss--sem-montagem-de-lakehouses).

Referência a seguir: `notebooks/dados_gov/dados_gov_GOLD.ipynb` — todas as dims, factos e pontes num único notebook, cada um escrito por `.save(path)`.

**Contra-exemplos no repo, a não copiar** (legado, anteriores a esta regra):

| Notebook | Violação |
|---|---|
| `notebooks/calamidade/page_apoio_calamidade_dims_nb (1).ipynb` | `create schema` + `saveAsTable("apoio_calamidade.dim_*")` |
| `notebooks/gapminder/gapminder_BRONZE.ipynb` (célula 6) | `.option("path", abfss_path).saveAsTable(table_name)` |

**Status:** implementado nos pipelines actuais excepto nos dois contra-exemplos acima. Verificado mecanicamente pelo `revisor-pipeline` e pela [Checklist Bronze](checklists/checklist-bronze.md).

---

### Criar a tabela e escrever nela são operações diferentes

**Quem cria não escreve.** A criação corre uma vez, à parte, e tem de ser repetível sem
destruir nada. A escrita corre a cada execução do pipeline. Misturar as duas numa célula é o
que transforma um `Run All` distraído numa tabela apagada.

**Criação — idempotente, nunca `overwrite`:**

```python
if not DeltaTable.isDeltaTable(spark, path):
    df_vazio.write.format("delta").mode("errorifexists").save(path)
```

As duas peças não são redundantes. A guarda trata do caso normal: a tabela existe, não se faz
nada. O `errorifexists` trata do caso estranho: o path existe mas não é uma tabela Delta
(sobras de uma execução falhada), a guarda deixa passar e a escrita rebenta em vez de escrever
por cima. Falha alto, que é o que se quer num DDL.

`mode("overwrite")` numa célula de criação escreve um DataFrame vazio por cima do que lá
estiver — uma tabela com 5 000 linhas fica com zero, sem erro nem aviso. O
`option("overwriteSchema", "true")` agrava: sem ele um `overwrite` com schema diferente falha e
obriga a olhar; com ele, aceita em silêncio.

**Escrita — três modos, e quando se usa cada um:**

| Modo | Quando |
|---|---|
| `mode("append")` | tabelas de histórico ou de factos, onde cada execução acrescenta |
| `delete` da fatia + `mode("append")` | substituir um subconjunto sem tocar no resto (por origem, por entidade, por período) |
| `mode("overwrite")` + `overwriteSchema` | **só** tabelas reconstruídas de raiz a cada execução, onde apagar é o objectivo |

**V-Order.** É uma optimização de escrita do Parquet que troca tempo de escrita por tempo de
leitura. Está **desligada por defeito** nos workspaces Fabric novos, e liga-se na sessão com
`spark.sql.parquet.vorder.default`. A chave antiga `spark.sql.parquet.vorder.enable` foi
**removida** no Runtime 1.3 — código que ainda a use não está a ligar nada, e não dá erro
([documentação](https://learn.microsoft.com/en-us/fabric/data-engineering/delta-optimization-and-v-order)).
Faz sentido nas tabelas Gold, que são lidas muitas vezes; não faz nas de ingestão.

O `delete` + `append` está detalhado em [Camada Gold](camada-gold.md) e é o padrão em produção
para tabelas alimentadas por várias fontes. **Não se usa `MERGE INTO`** — não há uma única
ocorrência no repositório, e o `delete` + `append` resolve o mesmo com menos peças.

**Acrescentar uma coluna a uma tabela que já tem dados** é operação própria, nunca recriar:

```python
spark.sql(f"ALTER TABLE delta.`{path}` ADD COLUMNS (nova_coluna STRING)")
```

Em alternativa, `option("mergeSchema", "true")` num `append`. Recriar a tabela para lhe
acrescentar uma coluna apaga o histórico.

---

## Orquestração Baseada em Metadados (Metadata-Driven)

Privilegiar pipelines genéricos e reutilizáveis: uma pipeline master que entende as suas transformações a partir de metadados (JSON, tabelas SQL), em vez de uma pipeline por transformação.

**Status parcial:**
- Os runners da geração actual lêem a configuração de um ficheiro JSON no Lakehouse, via `read_config_pipelines()`. Alterar um pipeline é editar o JSON, não o notebook
- Os runners da geração antiga ainda têm a configuração escrita no próprio notebook ([ver Orquestração](orquestracao-runner.md#as-duas-gerações-de-runner))
- **Trabalho futuro:** passar do ficheiro JSON para tabelas Delta, para se poder consultar e alterar a configuração por SQL

Três regras que já se sabem antes de o fazer, e que se aplicam a qualquer configuração guardada
numa tabela. Nenhuma está implementada:

1. **Actualizar é `delete` da fatia + `append`**, nunca `overwrite` da tabela toda. Assim actualiza-se um subconjunto de fontes sem tocar no resto.
2. **Desligar uma fonte é `enabled = false`, nunca apagar a linha.** Apagar quebra a ligação entre o histórico de execuções e a configuração que o produziu.
3. **A chave da configuração viaja com o registo.** Quem processa lê a configuração ao arrancar e leva a chave consigo até à escrita. Tentar reconciliar no fim, por nome, não funciona.

---

## Controlo de Versão — DevOps

As lógicas dos workloads não devem estar presas dentro da plataforma PaaS/SaaS — devem estar num repositório de código com integração Git.

- Utilizar as integrações Git do Fabric para manter um ambiente de DevOps
- Notebooks versionados em Git, não apenas no workspace Fabric
- Esta wiki também deve estar num repositório Git (Azure DevOps)

**Status:** a implementar — os notebooks e esta documentação devem ser migrados para Git com pipeline CI/CD.

---

## Observabilidade Agnóstica

Implementar um wrapper de logging transversal a todos os notebooks, que guarda eventos num repositório centralizado independente da plataforma.

Os campos que interessam à leitura dos logs (schema completo em [ELogger](bibliotecas-comuns.md#schema-da-tabela-de-logs)):

| Campo | Descrição |
|---|---|
| `correlation_id` | ID único da execução do pipeline |
| `pipeline_id` | Nome do pipeline |
| `notebook` | Notebook que gerou o evento |
| `step_name` | Passo dentro do notebook |
| `level` | `info`, `warn`, `error`, `dump` |
| `message` | Mensagem do evento |
| `s_from_start` | Segundos desde o início da execução |
| `timestamp` | UTC |

Os logs ficam em Delta Lake (`common_lh_observability`), consultáveis via SQL em qualquer ferramenta — não dependem do Fabric para ser lidos.

**Status:** implementado — ver [ELogger](bibliotecas-comuns.md#dap_eloggerpy).

---

## Modelo Semântico Agnóstico

Aproximar a camada Gold ao modelo semântico, de forma a que o modelo Power BI seja o mais fino possível (cálculos DAX mínimos, métricas já produzidas em Gold).

Balancear:
- Métricas já calculadas em Gold (portáveis, reutilizáveis por qualquer ferramenta BI)
- Funções DAX específicas do Power BI (potência analítica, já adoptadas na ARTE)

**Status:** implementado nas tabelas Gold actuais. A definição do equilíbrio correcto é caso-a-caso.

---

## Governação Portável

Implementar uma estratégia de governação que não dependa de uma ferramenta específica. Utilizar os **metadados do Delta Lake** como fonte de verdade para governação, de forma a que ferramentas externas (Microsoft Purview, Unity Catalog, Apache Atlas/Amundsen, etc.) possam consumir esses metadados sem integração proprietária.

**Status:** a implementar — os dados estão em Delta (metadados disponíveis), mas não existe ainda uma estratégia formal de governação configurada.

---

[← Visão Geral](visao-geral.md) · [Home](../Home.md)
