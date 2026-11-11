---
name: fabric-arte
description: Como escrever, rever e corrigir notebooks, configs de pipeline, Lakehouses e modelos semânticos Microsoft Fabric no repositório arte-data-platform-governance da ARTE (arquitectura Medallion Bronze/Silver/Gold, runner metadata-driven, Global Libs dap_*, ELogger). Usar sempre que a tarefa toque em engenharia/, negocio/, shared/, cicd/ ou docs/metodologia, ou em qualquer ficheiro notebook-content.py, .platform, config JSON de pipeline ou item Fabric.
---

# Trabalhar com o Microsoft Fabric na ARTE

Este repositório é a fonte de verdade da plataforma de dados da ARTE em Microsoft Fabric. Os
workspaces Fabric estão ligados ao Git e vão buscar o código aqui. **Tudo o que escreves aqui
chega a um workspace real.** Lê esta skill inteira antes de criar ou alterar um notebook.

A documentação completa está em `docs/` (começar por `docs/Home.md`). Esta skill resume as regras
que não se podem violar e diz onde ler o detalhe. Em caso de dúvida, a documentação em
`docs/metodologia/` e as checklists em `docs/metodologia/checklists/` prevalecem.

---

## 1. Antes de começar: perguntas que tens de fazer a ti próprio

1. **Em que workspace vive isto?** Uma pasta do repositório = um workspace Fabric, com o mesmo nome.
2. **É engenharia ou negócio?** Engenharia só tem notebooks e pipelines. Negócio tem Lakehouse Gold, modelo semântico e relatórios.
3. **Que camada é?** Bronze (cópia fiel), Silver (normalização), Gold (grão, chaves, KPIs). Ver secção 6.
4. **A fonte já é ingerida por outro pipeline?** Se o schema `<fonte>` já existe no Lakehouse de Bronze/Silver, lê-se de lá. Nunca reingerir.
5. **Há um exemplar para copiar?** Os de referência estão na secção 11. Parte sempre de um exemplar da geração actual, nunca de um notebook legado.

Se a tarefa exige um GUID, id de workspace, id de Lakehouse ou nome de segredo que não te foi
dado, **não o inventes nem o copies de outro pipeline "por analogia"**. Escreve
`<PREENCHER: descrição do valor>` e diz ao utilizador o que falta.

---

## 2. Organização dos workspaces (as sete regras)

| # | Regra |
|---|---|
| 01 | Duas camadas: **engenharia** (só notebooks e pipelines) e **negócio** (Lakehouse Gold, modelo master, relatórios). |
| 02 | Um workspace por área de negócio, criado quando a área o pede. |
| 03 | Engenharia: só a equipa de dados; developers são Contributor; Admin só quem gere a ligação Git. Negócio: equipa de dados Admin, 2–3 autores Contributor, restantes Viewer/app. |
| 04 | `_dev` + produção só quando várias pessoas mexem ao mesmo tempo ou uma avaria tem visibilidade pública. |
| 05 | Tudo em capacidade F64. Nada em capacidade partilhada. |
| 06 | **Atalho, nunca cópia.** Dados de outro workspace chegam por shortcut. Workspace que só publica relatórios usa ligação live ao modelo existente, não cria modelo próprio. |
| 07 | Nomes: engenharia `e_<produto>_ws` (prd) / `e_<produto>_dev_ws` (dev); negócio `<area>_ws`; transversais `e_common_ws`, `e_config_ws`. **Produção nunca leva sufixo de ambiente.** |

Workspaces especiais: `arte_insights_ws` (Lakehouse só com atalhos, um modelo master) e
`cto_office_ws` (só relatórios, ligação live).

---

## 3. Estrutura do repositório

```
docs/                     documentação (nenhum workspace lhe toca)
cicd/                     validações automáticas (validate_checklists.py, release notes)
.github/workflows/        GitHub Actions (validação nocturna, sync de workspaces, Teams)
shared/                   workspaces transversais (e_common_ws, e_config_ws)
engenharia/<dominio>/<workspace>/   notebooks e pipelines
negocio/<dominio>/<workspace>/      Lakehouse Gold, modelos semânticos, relatórios
```

Domínios actuais: `public_sector`, `smartcities`.

Regras da estrutura:

1. **Uma pasta por workspace, com o nome exacto do workspace.** Sem excepções.
2. **As camadas Bronze/Silver/Gold não são pastas do repositório.** Vivem dentro do workspace, em subpastas por produto (ex.: `e_equipa_operacao_ws/interoperabilidade/pi/`).
3. **Não mudes o nome nem o caminho de uma pasta de workspace já ligada.** O caminho não é editável depois da ligação; mudar obriga a desligar e voltar a ligar.
4. **Não renomeies a pasta de um item** (`X.Notebook` → `Y.Notebook`). Para o Fabric isso é apagar um item e criar outro: perde-se o `logicalId`, o agendamento e as referências.
5. Os `Readme.md` com "This is an auto-created file" são criados pelo Fabric. Não apagar.

### Formato dos itens Fabric no Git

Cada item é uma pasta `<displayName>.<Tipo>`:

| Tipo | Ficheiros |
|---|---|
| `.Notebook` | `notebook-content.py` + `.platform` |
| `.Lakehouse` | `lakehouse.metadata.json`, `shortcuts.metadata.json`, `alm.settings.json`, `.platform` |
| `.DataPipeline` | `pipeline-content.json` (JSON ARM) + `.platform` |
| `.SemanticModel` | `definition/*.tmdl`, `definition.pbism`, `.platform` |
| `.Report` | `definition/` (PBIR: `pages/`, `visuals/`), `definition.pbir`, `.platform` |

O `.platform` tem `metadata.type`, `metadata.displayName` e `config.logicalId`.
**Nunca alteres nem geres um `logicalId` à mão num item que já existe.** Para um item novo,
cria um GUID novo e único. Referências entre itens do mesmo workspace usam o `logicalId` e
`workspaceId` a zeros (`00000000-0000-0000-0000-000000000000`); não substituir por ids reais.

Nos ficheiros `.schedules` não se mexe: o agendamento é do ambiente, não do código.

### O ficheiro `notebook-content.py`

Este é o formato em que os notebooks estão no repositório (não `.ipynb`). Estrutura:

```python
# Fabric notebook source

# METADATA ********************

# META {
# META   "kernel_info": {
# META     "name": "synapse_pyspark"
# META   },
# META   "dependencies": {}
# META }

# CELL ********************

# MAGIC %%configure -f
# MAGIC {
# MAGIC   "defaultLakehouse": { "name": "...", "id": "...", "workspaceId": "..." }
# MAGIC }

# METADATA ********************

# META {
# META   "language": "python",
# META   "language_group": "synapse_pyspark"
# META }

# PARAMETERS CELL ********************

main_set = { ... }
running_local = True
log_correlation = {}

# METADATA ********************
# ...
# CELL ********************
# ...código...

# MARKDOWN ********************

# # Título de uma célula markdown
```

Regras do formato:

- A primeira célula de código é sempre o `%%configure -f` (linhas `# MAGIC`).
- A célula de parâmetros é marcada com `# PARAMETERS CELL ********************`. Sem esse marcador o Fabric não sabe onde injectar os valores do runner.
- Cada `# CELL` é seguida do seu bloco `# METADATA` com `language`/`language_group`.
- `"dependencies": {}` no topo fica **sempre vazio**. Não afixar Lakehouses por aqui.
- Se tiveres mesmo de produzir `.ipynb`: o `source` de cada célula é uma **lista de linhas** (nunca string única), e células `markdown` não têm `execution_count` nem `outputs`.

---

## 4. Fluxo de trabalho Git

**No portal do Fabric corre-se e consulta-se. Escreve-se no VS Code.**

| Branch | Workspace ligado | Quem escreve |
|---|---|---|
| `main` | Produção de cada produto | Ninguém directamente. Só por PR aprovada |
| `dev` | Desenvolvimento, nos produtos que o têm | Quem desenvolve, sem revisão |
| `dev-<workspace>` | Branch do workspace (ex.: `dev-e_giap_ws`) | O Fabric; sincronizada para `dev` por Action |
| Branch de tarefa | Nenhum | Quem a criou; apagada após merge |

Regras para o agente:

1. **Todo o trabalho começa numa branch de tarefa**, criada a partir da branch ligada ao workspace. PR para `main` (ou `dev`, se o produto tiver workspace de dev).
2. **Nunca fazer push directo para `main`.** Há um alerta para o Teams (`notifyteams.yml`) quando acontece.
3. O merge na `main` dispara a actualização do workspace via API `updateFromGit`. Não há aprovação dentro do Fabric: **a revisão da PR é o único ponto de controlo**.
4. **O Git guarda código, não dados.** Um `revert` repõe o notebook, mas não repõe uma tabela Gold que o notebook apagou ou reescreveu. Trata qualquer alteração a escritas (`overwrite`, `delete`) com o máximo cuidado e assinala-a na descrição da PR.
5. `releasenotes.md` é gerado automaticamente pelas Actions. Não editar à mão.
6. **Correr um notebook escreve nos dados reais.** Os paths vêm da configuração, não de onde se corre. Para experimentar, aponta-se a configuração para destinos de teste.

---

## 5. Regras invioláveis de código

Estas regras são verificadas por `cicd/validate_checklists.py`. Um notebook que as viole não passa.

### 5.1 Paths e Lakehouses

- **Nunca hardcodar** paths de Lakehouse, workspaces, GUIDs, contentores, credenciais ou nomes de tabela que variem por ambiente. Tudo chega pelo `main_set`.
- Paths sempre em **ABFSS**: `abfss://<workspace_guid>@onelake.dfs.fabric.microsoft.com/<lakehouse_guid>/Tables/<schema>/<tabela>`.
- **GUID ou nome, nunca os dois.** O sufixo `.Lakehouse` só acompanha o nome. `<guid>.Lakehouse` dá `400 Bad Request` em runtime.
- `/lakehouse/default/` só é permitido para `/lakehouse/default/Files/Global_Libs`. Nunca para dados.
- Valores com `abfss://` ou GUID só podem aparecer na célula de parâmetros (valores por defeito para correr à mão). Fora dela o validador falha.

```python
# ✅
lh_gold = main_set["lh_gold"]
path = f"{lh_gold}/Tables/{schema}/dim_entity"

# ❌
path = "abfss://33f78fb2-...@onelake.dfs.fabric.microsoft.com/abcd-...Lakehouse/Tables/dim_entity"
```

### 5.2 Escrita Delta: por path, nunca pelo catálogo

```python
# ✅ única forma permitida
df.write.format("delta").mode("overwrite").save(path)

# ❌ todas proibidas (dão DELTA_TABLE_LOCATION_MISMATCH na segunda execução)
df.write.saveAsTable("schema.tabela")
df.write.format("delta").option("path", path).saveAsTable("tabela")
df.writeTo("tabela")
spark.sql(f"CREATE TABLE t USING DELTA LOCATION '{path}'")
```

- **Zero `MERGE INTO`.** Usa `delete` da fatia + `append`.
- **Criar e escrever são operações separadas.** Criação idempotente, nunca `overwrite`:
  ```python
  if not DeltaTable.isDeltaTable(spark, path):
      df_vazio.write.format("delta").mode("errorifexists").save(path)
  ```
- Acrescentar uma coluna a uma tabela com dados: `ALTER TABLE delta.\`{path}\` ADD COLUMNS (...)` ou `mergeSchema` num `append`. **Nunca recriar a tabela.**
- `option("overwriteSchema", "true")` **só** onde o schema muda mesmo entre execuções. Por defeito troca uma falha visível por uma perda silenciosa de colunas.
- V-Order: a chave é `spark.sql.parquet.vorder.default`. A antiga `...vorder.enable(d)` foi removida no Runtime 1.3 e não faz nada. Faz sentido em Gold, não em ingestão.

| Modo | Quando |
|---|---|
| `append` | Histórico/factos que acumulam |
| `delete` da fatia + `append` | Substituir um subconjunto (por `origem`, entidade, período) |
| `overwrite` | **Só** tabelas reconstruídas de raiz a cada execução |

### 5.3 A célula `%%configure`

```
%%configure -f
{
  "defaultLakehouse": {
    "name": "<PREENCHER: nome do Lakehouse de suporte>",
    "id": "<PREENCHER: id>",
    "workspaceId": "<PREENCHER: workspaceId>"
  }
}
```

- Primeira célula de código, obrigatória, com `name`, `id` e `workspaceId`.
- Aponta para o **Lakehouse de suporte** (o que tem `Files/Global_Libs`, ex.: `common_lh`), **nunca** para o Lakehouse de dados do pipeline.
- **Todos os notebooks de um pipeline declaram o mesmo Lakehouse de suporte.** Um filho com Lakehouse diferente do runner é bloqueado.
- **Proibido** timeout de sessão: nem `sessionTimeoutInSeconds`, nem `conf` → `spark.fabric.notebook.session.timeout`. Uma sessão parada é cobrada como a trabalhar (causa do throttling de 29/10/2026).
- Se os ids não te foram dados, ficam `<PREENCHER: ...>`. Nunca os copies de outro pipeline.

### 5.4 Segredos

- Nunca hardcodar passwords, SAS tokens, connection strings ou API keys. Sempre `read_kv_secret(main_set[...key_vault], main_set[...secret_to_read])`.
- **Nunca logar** o valor de um segredo nem passá-lo ao ELogger.
- URL do Key Vault e nome do segredo chegam pelo `main_set`.

### 5.5 Outras

- Código novo usa `notebookutils`, não `mssparkutils` (renomeado, será retirado).
- Sem Data Factory Gen2, Dataflow Gen2 ou actividades visuais. **Thin pipelines, fat notebooks.** Única excepção: fontes atrás do gateway (Tipos 3, 4 e 5 da secção 6.1).
- Sem comentários `TODO`/`FIXME` nem `# Pergunta: ...` por resolver; sem células de teste esquecidas (ex.: `23/0`).

---

## 6. As camadas

### 6.1 Bronze — cópia fiel da fonte

Não transforma nada. Só acrescenta colunas de proveniência quando o destino é tabela.

| Fonte | Destino |
|---|---|
| Ficheiros (CSV, XLSX, JSON em blob) | `Files/<folder_destino>/`, tal como vieram |
| Resposta de API (JSON) | `Tables/<schema>/<tabela>` em Delta, estrutura aninhada preservada |

Os cinco tipos de ingestão:

| Tipo | Fonte | Onde corre a chamada |
|---|---|---|
| 1 | Azure Blob (consume-and-delete): SAS do Key Vault → `list_blobs_via_rest` → `read_blob_via_rest` → `upload_from_requestes_file_in_chunks` (4 MB) → `delete_blob_via_rest` **só após upload confirmado** | Notebook |
| 2 | REST API pública: retry com backoff, sessão `requests` reutilizada, paginação pelo `total`, parâmetros de rede no `main_set` | Notebook |
| 3 | API interna via VPN/PBIGW com OAuth2 | `WebActivity` de Fabric Pipeline, disparada pelo notebook com `sempy.fabric.FabricRestClient` |
| 4 | API interna via VPN/PBIGW com Basic Auth | Igual ao 3, `auth_header` passado com `secureInput: true` |
| 5 | Base de dados on-premises | Fabric CopyJob, disparado e acompanhado por REST. Não há watermark exacto; usar janela deslizante + dedupe na Silver |

O JSON de uma Fabric Pipeline (formato ARM) **não é** o config do runner. Não os confundir.

**O Bronze devolve sempre `new_data`** explícito. O runner lê só essa chave para decidir se
corre o resto. Os acumuladores que alimentam o `result` são declarados **fora** dos ciclos.

```json
{ "status": "OK", "errorId": 0, "new_data": 1, "files_copied": [...], "resumo": "5/5 ficheiros gravados" }
```

### 6.2 Silver — normalização

- Lê o Bronze por ABFSS. Escreve `mode("overwrite")` (volume grande: `delete` da fatia + `append`).
- Normalização declarativa por `schema_map` de `{input, output, type}`, aplicada por uma função genérica. `input` aceita notação com ponto para JSON aninhado; um `input` inexistente gera coluna `null` com aviso, não rebenta.
- Contrato dos casts: **um cast que falha devolve `null`, nunca excepção**; um `type` desconhecido avisa. Tipos: `string`, `int`, `long`, `double` (PT `1.234,56` e EN `1,234.56`), `boolean`, `date`, `timestamp` (lista de formatos no `schema_map`), `raw` (sem cast). As funções de cast vivem dentro de cada notebook e **variam entre pipelines**: lê as do notebook de onde partes, não assumas.
- Strings `"NULL"` (qualquer caixa) → `null` real.
- Colunas de auditoria mínimas: `origem` (string, a chave que a Gold usa no `delete`) e `dt_ingest` (timestamp). Só se o Bronze for ficheiro com data no nome: `ficheiro_origem` e `data_ficheiro`.
- Ficheiros que não se lêem por ABFSS (xlsx, GeoJSON): `spark.read.format("binaryFile").load(p).select("content").head()["content"]` e `BytesIO`. Nunca copiar para disco nem ler de `/lakehouse/default/`.
- **Duas zonas:** `silver_origem.<fonte>` (1:1 com a origem, a regra) e `silver_negocio.<entidade>` (opcional; só quando **duas ou mais** tabelas Gold precisam da mesma junção). `silver_negocio` mantém o grão: sem chaves substitutas, sem agregação, sem KPIs.
- **Cuidado com `new_data`:** uma Silver que devolva `new_data: 0` faz saltar a Gold. Devolve `1` ou omite a chave.

### 6.3 Gold — grão, chaves e KPIs

- Lê Silver por `lh_silver`, escreve por `lh_gold` (pode ser outro workspace; é só um path).
- **Star schema num só notebook**: dims antes dos factos, um step por tabela, cada step guardado por `if "<step>" in steps:` com `steps = main_set["task_config"].get("running_steps", [])`.
- Escolher o padrão pelo que a tabela é: reconstruída de raiz → `overwrite`; várias fontes reprocessáveis → `delete` por `origem` + `append`; histórico → `append`.
- Joins com mapeamentos são sempre `left`, com `F.coalesce(..., F.lit("<default>"))`. **Nunca perder linhas num join.**
- Último step: `meta_execucao` (`pipeline_id`, `ambiente`, `linhas_fact`, `dt_carga`), com `replaceWhere` por `pipeline_id` se o schema for partilhado.
- Somas e contagens a grão fixo calculam-se no Gold. Rácios, distinct counts e comparações temporais ficam como medidas DAX.
- O acumulador `results` é declarado fora dos `try` e passado ao `handleError` em cada `except`.
- O refresh do modelo semântico **não** é feito no notebook Gold: é uma task `sm_refresh` no runner.

```json
{ "status": "OK", "results": [ {"step": "dim_entity", "rows": 1234}, {"step": "fact_x", "rows": 56789} ] }
```

### 6.4 Fontes partilhadas

Cada fonte é ingerida uma só vez. Usada por dois ou mais pipelines → pipeline próprio com o nome
da fonte, só ingestão. Desde o primeiro dia a fonte tem a sua pasta `Files/<fonte>/` e o seu
schema `<fonte>`. Consumidores lêem a última camada da fonte; falta uma tabela → acrescenta-se ao
config de quem ingere. Lógica de negócio fica no pipeline de cada negócio.

---

## 7. Anatomia obrigatória de um notebook de camada

| # | Célula | Conteúdo |
|---|---|---|
| 0 | `%%configure -f` | Lakehouse de suporte (secção 5.3) |
| 1 | `PARAMETERS CELL` | `main_set = {...}` com valores por defeito, `running_local = True`, `log_correlation = {}` |
| 2 | imports | standard e PySpark |
| 3 | desempacotar | ver código abaixo |
| 4 | configuração | ler do `main_set` paths, `pipeline_id`, `group`, `nb`; derivar paths das tabelas |
| 5 | ELogger | novo se `running_local`, reconstruído de `log_correlation` se não |
| 6+ | lógica | um passo por célula, cada um em `try`/`except` com `errorId` próprio (10, 20, 30...) |
| última | resultado | `result`, `elogger.log_event_Finish()`, `notebookutils.notebook.exit(json.dumps(result))` se `not running_local` |

```python
# célula 3
import sys, json
if type(main_set) == type(""):
    main_set = json.loads(main_set)
    log_correlation = json.loads(log_correlation)

sys.path.append("/lakehouse/default/Files/Global_Libs")
from dap_ELogger import ELogger
from dap_fabric_config_tools import handleError
```

```python
# célula 4 — leitura do main_set montado pelo build_call_nb_params
try:
    lh_silver   = main_set["lh_silver"]                       # campos de allTasks na raiz
    lh_gold     = main_set["lh_gold"]
    pipeline_id = main_set["pipeline_info"]["pipeline_id"]
    ws          = main_set["pipeline_info"]["workspace"]
    logLevel    = main_set["pipeline_info"]["log_level"]
    group       = main_set["group_context"]["group_name"]
    nb          = main_set["task_config"]["nb"]
    steps       = main_set["task_config"].get("running_steps", [])
except Exception as e:
    handleError(10, e, running_local, {"step": "config"})
```

```python
# célula 5
if running_local:
    elogger = ELogger(workspace=ws, pipeline_id=pipeline_id, group=group, nb=nb,
                      logLevel=logLevel, printOnScreen=True)
    elogger.log_event_start(message=f"group:{group}")
else:
    elogger = ELogger(correlation=log_correlation, workspace=ws, nb=nb)
```

```python
# células de lógica
try:
    df = normalizar_colunas(df, schema_map)
except Exception as e:
    handleError(40, e, running_local, results)
```

```python
# última célula
result = {"status": "OK", "errorId": 0, "new_data": 1, "resumo": f"{n} linhas escritas"}
elogger.log_event_Finish()
if not running_local:
    notebookutils.notebook.exit(json.dumps(result))
```

### `handleError`

- Vem de `dap_fabric_config_tools`. **Não devolve nada**: sai do notebook (`notebook.exit`) em produção, levanta `KeyError` em local.
- Assinatura: `handleError(step_number, e, running_local, results)`.
- **Proibida** a forma antiga `errorState, errorId = handleError(...)` e guardas `if errorId == 0:`.
- Chama-se dentro do `except`. **Nunca** embrulhar o `handleError` nem o `notebook.exit` noutro `try` com `except` largo: o `exit` lança uma excepção que seria engolida.
- `errorId` é local ao notebook. Não há tabela de códigos partilhada.

### ELogger — custo

Cada `elogger.log_event_*` é um job Spark + commit Delta noutro workspace: **~8 s fixos por
chamada**. Caso real: 928 logs num ciclo = 2h11m para 11 s de trabalho.

- **Nunca** `log_event_*` dentro de `for`/`while`. Acumula numa lista e loga **uma vez** no fim; dentro do ciclo, só por excepção.
- Erros críticos: `log_event_error(..., logLevel=1)`. Debug: `logLevel=5`.
- `print` de um filho não aparece no runner. O que o runner deve mostrar vai no campo `resumo` do `result`.

Logs em `e_common_ws` / `common_lh_observability`, tabela `ELogger.pipeline_logs`, por `correlation_id`.

---

## 8. O runner e a configuração

### 8.1 O runner (geração actual)

Reconhece-se por declarar `nb_settings` (não `main_set`) na célula de parâmetros.

```python
# PARAMETERS CELL
local_updates = []
nb_settings = {"config_file": "<nome_do_pipeline>", "local_updates": local_updates}
running_local = True
log_correlation = {}
```

Sequência obrigatória (copiar de `pi_runner`):

1. Desempacotar `nb_settings` e `log_correlation` com `json.loads`; imports de `dap_ELogger`, `dap_fabric_semantic_tools.semantic_model_refresh_wait` e `dap_fabric_config_tools.{read_config_pipelines, handleError, build_call_nb_params, update_fabric_metadata, run_notebook_with_params, get_execution_plan}`.
2. `main_set = read_config_pipelines(nb_settings["config_file"])` em `try`/`except` + `handleError`.
3. **Resolver o ambiente**: `ws_actual = notebookutils.runtime.context["currentWorkspaceId"]`; parar se não estiver em `main_set["ambientes"]` ou se tiver `<PREENCHER`; copiar para `allTasks` todas as chaves do ambiente menos `nome`; `main_set["workspace"] = ws_actual`; `main_set["allTasks"]["ambiente_nome"] = ambiente["nome"]`. **Nunca assumir um ambiente por omissão.**
4. Aplicar `local_updates` com `update_fabric_metadata` (o bloco existe mesmo com lista vazia).
5. ELogger com o `ws_actual`, nunca o `workspace` do JSON.
6. `plan = get_execution_plan(main_set)` e imprimir o plano.
7. Ciclo: `new_data = 0` antes; SKIP se `group_enabled != 1 or task_enabled != 1`; SKIP se `new_data != 1 and run_withouth_new_data != 1`; `type == "notebook"` → `build_call_nb_params` → `run_notebook_with_params(nb, timeout, {"main_set": json.dumps(...), "running_local": False, "log_correlation": json.dumps(elogger.get_correlation())})`; `handleError` se `status == "ERROR"`; `new_data = run_result.get("new_data", 1)`.
8. `type == "sm_refresh"` → sincronizar o SQL endpoint do `lh_gold` (`refreshMetadata`) **antes** de `semantic_model_refresh_wait(dataset, ws, wait_s, times)` com `wait_s`/`times` explícitos (os defaults dão 25 s). O refresh é sempre `full`.
9. `elogger.log_event_Finish()`.

### 8.2 O ficheiro de configuração

Vive em `Files/configs/pipelines/<config_file>.json` no Lakehouse de configuração (diferente do
de suporte). **Nunca se edita à mão.** É escrito por um notebook gerador
`<config_file>_e_config_lh_json`, versionado no Git:

```python
from dap_fabric_config_tools import dap_fabric_config_pipelines_tools_path
notebookutils.fs.put(f"{dap_fabric_config_pipelines_tools_path}/<config_file>.json",
                     json.dumps(cfg, indent=4), overwrite=True)
```

**Nunca** `Files/configs/pipelines/...` relativo: grava no Lakehouse por defeito da sessão, sem erro.

```json
{
  "workspace": "<workspace_id>",
  "pipeline_id": "<pipeline_id>",
  "log_Level": { "logLevel": 5, "printOnScreen": true },
  "allTasks": { "folder_destino": "Files/<dominio>/ingest", "schema_silver": "<schema>" },
  "ambientes": {
    "<PREENCHER: id do workspace de dev>": { "nome": "dev", "lh_bronze": "abfss://...", "lh_silver": "abfss://...", "lh_gold": "abfss://..." },
    "<PREENCHER: id do workspace de prd>": { "nome": "prd", "lh_bronze": "abfss://...", "lh_silver": "abfss://...", "lh_gold": "abfss://..." }
  },
  "groups": [
    {
      "order": 1, "group": "<pipeline>-bronze", "enabled": true, "run_withouth_new_data": true,
      "tasks": [
        { "order": 1, "nb": "<pipeline>_BRONZE", "type": "notebook",
          "movement_type": "external_to_bronze", "enabled": true, "timeout": 3600,
          "datasource": { } }
      ]
    },
    {
      "order": 4, "group": "<pipeline>-sm", "enabled": true, "run_withouth_new_data": false,
      "tasks": [
        { "order": 1, "nb": "", "type": "sm_refresh", "dataset_name": "<modelo>",
          "workspace_id": "<PREENCHER>", "enabled": true, "timeout": 3600 }
      ]
    }
  ]
}
```

Armadilhas que fazem tasks desaparecer **sem erro**:

- `enabled` tem de ser booleano JSON (`true`/`false`). A string `"True"` nunca iguala `1` e a task é saltada.
- **O primeiro grupo tem de ter `run_withouth_new_data: true`** (o runner começa com `new_data = 0`). Sim, o nome da chave tem o erro ortográfico `withouth`; não corrigir.
- `timeout` por task: **3600 por defeito, nunca acima**.
- Ids de workspace no bloco `ambientes` ficam `<PREENCHER: ...>` até o utilizador os dar.
- `movement_type` é informativo no runner actual. Valores canónicos: `external_to_bronze`, `bronze_to_silver`, `silver_to_gold`, `update_on_gold`. Não inventes variantes.
- Campos de negócio de uma task (ex.: `datasource`, `running_steps`, `schema`) acrescentam-se à task e chegam ao filho em `main_set["task_config"]`.
- Antes de ligar dev e prd ao Git, `local_updates = []` nos dois runners.

### 8.3 Lakehouses por ambiente

| Camada | dev | prd |
|---|---|---|
| Bronze e Silver | `d_arte_dev_lh` (`d_arte_dev_ws`) | `d_arte_lh` (`d_arte_ws`) |
| Gold | `d_arte_dev_lh` | Lakehouse do workspace de negócio (ex.: `equipa_operacao_lh`) |

Os Lakehouses de dados são partilhados por todos os produtos; cada produto separa-se pela pasta
em `Files/` e pelo schema em `Tables/`. Workspaces de engenharia não guardam dados.

---

## 9. Global Libs (`/lakehouse/default/Files/Global_Libs/`)

Compiladas por notebooks `common_nb_compile_<lib>` (só a célula que faz `write` conta). Sem
versionamento: alterar uma lib afecta todos os pipelines na execução seguinte. Depois de
republicar, reiniciar a sessão. Não se usam Environments do Fabric.

| Módulo | Funções principais |
|---|---|
| `dap_fabric_config_tools` | `handleError`, `read_config_pipelines`, `build_call_nb_params`, `update_fabric_metadata`, `run_notebook_with_params` (em erro levanta `KeyError` com JSON em `e.args[0]`), `get_execution_plan`, `dap_fabric_config_pipelines_tools_path` |
| `dap_ELogger` | `ELogger`, `log_event_start/Finish/info/warn/error/dump`, `log_rowCount`, `get_correlation`, `get_seconds` |
| `dap_fabric_semantic_tools` | `semantic_model_refresh`, `semantic_model_refresh_wait`, `semantic_model_refresh_request_history` |
| `dap_key_vault` | `read_kv_secret(vault_url, secret_name)` |
| `dap_azure_blob` | `list_blobs_via_rest`, `read_blob_via_rest`, `delete_blob_via_rest` |
| `dap_lake_file_tools` | `upload_from_requestes_file_in_chunks`, `upload_jsonl_to_abfss_in_chunks`, `list_files_by_date_mask_startingDate` (apesar do nome, **não** filtra por data), `extract_date_from_name`, `format_filename`, `get_files_by_mask`, `delete_abfss_files` (confirmar o valor devolvido) |
| `dap_json_validator` | `validate_payload(payload, bundle_path, modelo=None, ngsi_ld=True, spark=None, modelo_conjunto=None)` contra o Catálogo Nacional de Modelos de Dados, offline |

**Não usar** `list_abfss_files_by_pattern` (devolve sempre lista vazia). Usar `get_files_by_mask`.

Ficheiros em blob devem começar por data: `yyyyMMddHHmm_*` ou `yyyyMMdd_*`.

---

## 10. Normas de domínio

**PGU e Aviso de Dados** (`docs/metodologia/norma-pgu-aviso.md`), aplicada a notebooks em pastas
`pgu`, `aviso_dados`, `partilha_dados` ou com prefixo `partilha_dados`:

- Bronze: `Files/<dominio>/bronze/<entidade_id>/<conjunto_id>/{validado|rejeitado}/ano=<AAAA>/periodo=<codigo>/<periodo>_<entidade_id>_<conjunto_id>_<AAAAMMDDHHmm>.json` (metadados com `_md` antes da data de ingestão).
- Três tabelas comuns com coluna `dominio`: `partilha_dados_config`, `partilha_dados_processamentos`, `partilha_dados_erros`. O validador rejeita nomes antigos (`pgu_processamentos`, `aviso_dados_config`, ...).
- Seis estados: `CONFORME`, `NAO_CONFORME`, `MODELO_DESCONHECIDO`, `NAO_E_DCAT`, `JSON_INVALIDO`, `FONTE_INDISPONIVEL`. `estado` nunca vazio. Tudo o que não é `CONFORME` vai para `rejeitado/`.
- `correlation_id` vem de `log_correlation`, nunca gerado no notebook. `processamento_uuid` é `uuid4`; `cfg_uuid` é `uuid5(dominio + entidade_id + conjunto_id)`.
- Desligar uma fonte é `enabled = false`, nunca apagar a linha.
- Língua das colunas: português.

---

## 11. Exemplares a seguir e a evitar

**Seguir** (geração actual), em `engenharia/public_sector/e_equipa_operacao_ws/`:

- `interoperabilidade/pi/pi_runner.Notebook` — runner completo com bloco `ambientes` (célula 4) e `sm_refresh`
- `interoperabilidade/pi/pi_BRONZE.Notebook` — Bronze com `resumo`
- `interoperabilidade/pi/pi_SILVER.Notebook`, `pi_GOLD.Notebook` — Silver com `schema_map`; Gold com `running_steps`
- `interoperabilidade/pi/pi_bronze_writer.Notebook` — gravador chamado por Fabric Pipeline (sem ELogger, excepção documentada)
- `pi_operacao/pi_operacao_GOLD.Notebook` — `meta_execucao`

Exemplo de ponta a ponta documentado: `docs/metodologia/exemplo-dados-gov.md`.

**Não copiar**: runners da geração antiga (`main_set = {...}` com `groups` dentro, funções
`build_nb_ingest_params` etc.), notebooks com `saveAsTable`/`CREATE TABLE`, `%%configure` com
`conf` de timeout, `handleError` que devolve tuplo, `mssparkutils` em código novo.
Se encontrares um destes padrões numa tarefa, não o propagues; assinala-o ao utilizador.

---

## 12. Validar antes de entregar

Corre sempre o validador sobre o que alteraste:

```bash
python -m pip install bandit radon
git diff --name-only -z origin/main... > /tmp/changed.txt
python cicd/validate_checklists.py --files-from /tmp/changed.txt
# varrimento completo, só informativo:
python cicd/validate_checklists.py --allow-missing-configure --informational
```

Depois percorre a checklist da camada em `docs/metodologia/checklists/`
(`checklist-bronze.md` secção A aplica-se a **todos** os notebooks de camada;
`checklist-silver.md`, `checklist-gold.md`, `checklist-runner.md`).

Checklist rápida final:

- [ ] `%%configure -f` primeira célula, Lakehouse de suporte, sem timeout de sessão
- [ ] `# PARAMETERS CELL` com `main_set`/`nb_settings`, `running_local = True`, `log_correlation = {}`
- [ ] `json.loads` do `main_set`/`nb_settings` e do `log_correlation`
- [ ] Global Libs importadas de `/lakehouse/default/Files/Global_Libs`
- [ ] Nenhum `abfss://` nem GUID fora da célula de parâmetros; nenhum `<guid>.Lakehouse`
- [ ] Zero `saveAsTable`, `writeTo`, `CREATE TABLE`, `MERGE INTO`
- [ ] Modo de escrita escolhido pelo que a tabela é; `overwriteSchema` só se necessário
- [ ] `handleError` da biblioteca em cada `except`, nunca na forma de tuplo
- [ ] Nenhum `log_event_*` dentro de ciclos
- [ ] Bronze devolve `new_data`; Silver não devolve `0` por engano; Gold devolve `results`
- [ ] `notebookutils.notebook.exit(json.dumps(result))` quando `not running_local`
- [ ] Config: `enabled` booleano, `timeout` ≤ 3600, primeiro grupo com `run_withouth_new_data: true`, bloco `ambientes`
- [ ] Nenhum segredo no código ou nos logs
- [ ] Valores desconhecidos como `<PREENCHER: ...>`, nunca inventados
- [ ] Branch de tarefa e PR; descrição da PR assinala qualquer escrita destrutiva

---

## 13. Quando um pipeline falha (resumo do runbook)

1. Encontrar a execução: `SELECT correlation_id, MIN(timestamp), MAX(timestamp) FROM common_lh_observability.ELogger.pipeline_logs WHERE pipeline_id = '<p>' GROUP BY correlation_id ORDER BY 2 DESC`.
2. Ver onde parou: logs desse `correlation_id` por `timestamp`; procurar `level = 'error'`, último `step_name`, ausência de `Finish`.
3. Sem logs nenhuns → o import das Global Libs falhou (corrigir o `%%configure` **do runner**) ou o nome do `config_file` está errado.
4. Notebook que não correu sem erro → é SKIP: `enabled` false ou string, ou `new_data = 0` com `run_withouth_new_data = false`.
5. Re-executar só o que falhou: `local_updates` no runner, `running_steps` na Gold, ou notebook isolado com `running_local = True`.
6. Antes de repetir: Bronze consume-and-delete pode já ter apagado o blob; Gold `append` puro duplica linhas.
7. Execução muito lenta → procurar `log_event_*` dentro de ciclos.

Detalhe completo em `docs/metodologia/runbook.md`.
