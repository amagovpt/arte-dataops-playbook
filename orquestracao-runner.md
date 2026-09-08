# Orquestração — Runner

O runner é o único ponto de entrada de cada pipeline. Lê a configuração de um ficheiro JSON no Lakehouse, monta os parâmetros de cada notebook e invoca-os sequencialmente via `notebookutils.notebook.run(...)`.

---

## As Duas Gerações de Runner

Há dois padrões de runner em produção. Esta página descreve o **actual**. O antigo não está depreciado — está a correr — mas nenhum runner novo deve ser escrito assim.

| | Actual | Antigo |
|---|---|---|
| Configuração | ficheiro JSON no Lakehouse, lido por `read_config_pipelines` | `main_set = {...}` escrito à mão na célula `parameters` |
| Funções de orquestração | nas Global Libs (`dap_fabric_config_tools`) | definidas dentro do próprio notebook |
| Montagem de parâmetros | uma função, `build_call_nb_params` | uma função por camada (`build_nb_ingest_params`, `build_nb_silver_params`, `build_nb_gold`) |
| Ciclo de execução | percorre `get_execution_plan(main_set)` | percorre `groups`/`tasks` à mão |
| `movement_type` | não chega ao ciclo do runner | é o `if` que decide o ramo de execução |
| `new_data` | lido da chave `new_data` do JSON devolvido pelo notebook | derivado da contagem de `files_copied` do Bronze |

**Como reconhecer:** se a célula `parameters` do runner declara `nb_settings`, é o actual; se declara `main_set = {...}` com os `groups` lá dentro, é o antigo.

A distinção importa ao ler o resto desta wiki: várias afirmações só valem para uma das gerações, e daqui para a frente diz-se sempre qual.

---

## Os Dois Contratos de Configuração

O runner e os notebooks-filho não recebem a mesma coisa. É a distinção mais importante desta página.

| | Runner | Notebooks-filho |
|---|---|---|
| Variável na célula `parameters` | `nb_settings` | `main_set` |
| Conteúdo | `config_file` e `local_updates` | configuração já montada para aquela task |
| Origem | escrita à mão no notebook | injectada pelo runner em runtime |

### No runner

A célula de parâmetros do runner declara apenas onde está a configuração:

```python
nb_settings = {
    "config_file"   : "<nome_do_pipeline>",   # nome do ficheiro JSON, sem extensão
    "local_updates" : [],                     # sobreposições pontuais, ver abaixo
}
running_local = True
log_correlation = {}
```

O `main_set` **não** é declarado à mão no runner. É o que `read_config_pipelines` devolve:

```python
main_set = read_config_pipelines(nb_settings["config_file"])
```

### O ficheiro de configuração

```json
{
  "workspace"   : "<workspace_id>",
  "pipeline_id" : "<pipeline_id>",
  "log_Level"   : { "logLevel": 5, "printOnScreen": true },
  "allTasks"    : {
    "lh_bronze"      : "abfss://<workspace_guid>@onelake.dfs.fabric.microsoft.com/<lakehouse_guid>",
    "lh_silver"      : "abfss://...",
    "lh_gold"        : "abfss://...",
    "folder_destino" : "Files/<dominio>/ingest"
  },
  "groups": [ grupo_1, grupo_2 ]
}
```

O bloco `allTasks` reúne os campos partilhados por todas as tasks do pipeline. Bronze e Silver podem partilhar o mesmo Lakehouse; a Gold é habitualmente num Lakehouse separado.

### Onde vive o ficheiro

```
Files/configs/pipelines/<config_file>.json
```

Num Lakehouse fixo, cujo workspace e id estão **escritos no código** da `read_config_pipelines`. Duas consequências:

1. É o único valor operacional hardcoded do sistema. Mudar de Lakehouse de configuração obriga a recompilar a biblioteca, não a alterar um parâmetro.
2. **Não é o Lakehouse de suporte** declarado no `%%configure` dos notebooks — os GUIDs são diferentes. São dois Lakehouses distintos: um guarda as configurações dos pipelines, o outro as Global Libs.

### `local_updates`

Cada entrada sobrepõe um valor da configuração em runtime, sem editar o ficheiro JSON:

```python
local_updates = [
    {"group_order": 1, "task_order": 1, "key": "enabled", "new_value": False},
]
```

O runner aplica-as com `update_fabric_metadata` antes de montar o plano de execução.

---

## Estrutura de um Grupo

```python
{
    "order"                 : 1,          # ordem de execução (crescente)
    "group"                 : "nome-do-grupo",
    "enabled"               : True,       # False → grupo ignorado na totalidade
    "run_withouth_new_data" : True,       # False → skip se bronze não tiver novos dados
    "tasks"                 : [ tarefa_1, tarefa_2, ... ]
}
```

---

## Estrutura de uma Tarefa

```json
{
    "order"         : 1,
    "nb"            : "nome_do_notebook",
    "type"          : "notebook",
    "movement_type" : "external_to_bronze",
    "enabled"       : true,
    "timeout"       : 72000
}
```

`enabled` tem de ser booleano JSON (`true`/`false`). Uma string `"True"` nunca é igual a `1` na comparação que o runner faz, e a task passa a ser ignorada em silêncio, sem erro.

Numa task de refresh de modelo semântico (`type: "sm_refresh"`) o `nb` fica vazio (`""`) e acrescentam-se `dataset_name` e `workspace_id`.

Campos de negócio específicos da task, por exemplo um bloco `datasource`, acrescentam-se aqui e chegam ao notebook sem alterações às bibliotecas partilhadas.

---

## `type` e `movement_type`

São dois campos diferentes, e só um deles decide o que o runner faz.

### `type` — decide o ramo de execução

| `type` | O que o runner faz |
|---|---|
| `notebook` | Monta os parâmetros e invoca o notebook |
| `sm_refresh` | Chama `semantic_model_refresh_wait` com `dataset_name` e `workspace_id` |

### `movement_type` — informativo no runner actual

O `get_execution_plan` não devolve este campo, por isso o ciclo do runner actual não o vê sequer. Chega ao notebook-filho dentro de `task_config`, e é lá que é interpretado, se alguém o interpretar. Nos runners da geração antiga é o contrário: é o `movement_type` que decide o ramo de execução.

Os quatro valores canónicos, a usar num pipeline novo:

| `movement_type` | Camada | Intenção |
|---|---|---|
| `external_to_bronze` | Bronze | Copiar dados de uma fonte externa para o Lakehouse Bronze |
| `bronze_to_silver` | Silver | Ler Bronze, normalizar schema, escrever Silver |
| `silver_to_gold` | Gold | Ler Silver, harmonizar, escrever Gold |
| `update_on_gold` | Gold | Reler a Gold completa e enriquecer |

**Não existe validação deste campo, e nota-se.** Um levantamento a todos os notebooks e configs do
repositório dá **dez** valores distintos: aos quatro acima juntam-se `bronze_to_gold` e
`run_copyjob_pipeline` (pipelines sem Silver, alimentados por CopyJob), `update_dims`, e três
variantes inventadas por sufixo — `external_to_bronze_Delta`, `external_to_bronze_ppap`,
`bronze_to_silver_ppap`.

Nenhuma destas variantes dá erro, porque ninguém valida o campo. Nos runners da geração antiga são
lidas por um `if` escrito à mão nesse runner; no runner actual não são lidas de todo. Ao ler uma
configuração antiga, não assumir que o valor significa alguma coisa por si — procurar onde é lido,
se é.

---

## Lógica de Controlo de Fluxo

`get_execution_plan(main_set)` transforma a hierarquia de grupos e tasks numa lista plana, ordenada pelo `order` do grupo e depois pelo da task. O runner percorre essa lista:

```
Para cada entrada do plano:
  ├── Se group_enabled != 1 ou task_enabled != 1 → SKIP
  ├── Se new_data != 1 e run_withouth_new_data != 1 → SKIP
  ├── Se type == "notebook":
  │     → build_call_nb_params → run_notebook_with_params
  │     → se o resultado trouxer status "ERROR" → handleError
  │     → new_data = resultado.get("new_data", 1)
  └── Se type == "sm_refresh":
        → semantic_model_refresh_wait(dataset_name, workspace_id, ...)
```

`run_withouth_new_data = True` no grupo garante que o pipeline avança mesmo sem dados novos, útil quando um passo anterior pode ser ignorado mas os seguintes precisam de correr.

### `new_data`

O runner lê a chave `new_data` do JSON devolvido pelo notebook. **Não** a deriva de `files_copied` nem de nenhum outro campo.

Daí três consequências práticas:

1. Um notebook Bronze tem de devolver `new_data` explicitamente. Ver [Camada Bronze](camada-bronze.md#resultado-devolvido-ao-runner).
2. **Todas** as tasks de tipo `notebook` reescrevem `new_data`, não apenas as de ingestão, e o valor por omissão é `1`. Um notebook Silver que não devolva a chave repõe o fluxo em "há dados novos".
3. O runner inicia `new_data = 0` antes do ciclo. Por isso **o primeiro grupo tem de ter `run_withouth_new_data = true`**, senão é ignorado e nada corre.

---

## Construção de Parâmetros

Há uma única função, `build_call_nb_params(main_set, group_order, task_order)`, usada para qualquer task. Não existem funções distintas por camada.

O dicionário devolvido, que chega ao notebook-filho como `main_set`, tem esta forma:

| Chave | Conteúdo |
|---|---|
| campos de `allTasks` | promovidos à raiz, por exemplo `lh_bronze`, `lh_silver`, `lh_gold`, `folder_destino` |
| `pipeline_info` | `workspace`, `pipeline_id`, `log_level` |
| `group_context` | `group_name`, `run_withouth_new_data` |
| `task_config` | a task inteira, tal como está no JSON, incluindo os campos de negócio |

No notebook-filho, os valores lêem-se daí:

```python
lh_bronze   = main_set["lh_bronze"]
pipeline_id = main_set["pipeline_info"]["pipeline_id"]
group       = main_set["group_context"]["group_name"]
datasource  = main_set["task_config"]["datasource"]
```

---

## Anatomia de um Notebook de Camada

Todos os notebooks Bronze, Silver e Gold têm a mesma ordem de células. Não é convenção estética: cada passo depende do anterior.

| # | Célula | Conteúdo |
|---|---|---|
| 0 | `%%configure -f` | `defaultLakehouse` do Lakehouse de suporte. Tem de ser a primeira célula de código — ver secção seguinte |
| 1 | `parameters` (tag) | `main_set` com valores por defeito, `running_local = True`, `log_correlation = {}` |
| 2 | imports | bibliotecas standard e PySpark |
| 3 | desempacotar | `if type(main_set) == type(""): main_set = json.loads(main_set)`, depois `sys.path.append("/lakehouse/default/Files/Global_Libs")` e os imports `dap_*` |
| 4 | configuração | ler do `main_set` os paths ABFSS, `pipeline_id`, `group`, `nb`, e derivar os paths das tabelas |
| 5 | ELogger | instância nova se `running_local`, reconstruída de `log_correlation` se não |
| 6+ | lógica | um passo por célula, cada um em `try`/`except` com o seu `errorId` |
| última | resultado | montar o `result`, `log_event_Finish()`, e `notebookutils.notebook.exit(json.dumps(result))` se `not running_local` |

O runner tem o mesmo esqueleto, com `nb_settings` na célula 1 em vez de `main_set`, e a configuração a chegar do `read_config_pipelines`.

### `errorId`

Cada passo com `try`/`except` recebe um `errorId` próprio, crescente pela ordem das células (10, 20, 30, ...). O número não tem significado fora do notebook: serve para localizar em que passo a execução parou, lendo o JSON devolvido ao runner. **Não há tabela de códigos partilhada entre pipelines, e não deve haver** — as gamas listadas nas páginas de cada camada são exemplos do pipeline que as originou, não um contrato.

### O ficheiro `.ipynb`

Restrições observadas neste repositório ao importar notebooks para o Fabric. Não foram confirmadas em documentação da Microsoft, vêm de erros reais.

1. O `source` de cada célula tem de ser **lista de linhas**, nunca uma string única. É válido pela especificação nbformat, mas o Fabric rejeita o import com um erro genérico. Acontece tipicamente quando o `.ipynb` é editado por um programa depois de gerado.
2. Células `markdown` não podem ter `execution_count` nem `outputs` — são campos exclusivos de células `code`.
3. `metadata.dependencies`, no topo do ficheiro e fora das células, é o que o Fabric mostra como "Lakehouse afixado" no explorador. Não é o `%%configure`, e um não substitui o outro.

---

## A Célula `%%configure`

Todo o notebook deve começar com uma célula `%%configure -f` que declara o Lakehouse de **suporte** como `defaultLakehouse`. É esse Lakehouse que contém `Files/Global_Libs`, e nunca o Lakehouse de dados do pipeline.

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

Para prolongar o timeout da sessão, a chave documentada pela Microsoft é `sessionTimeoutInSeconds`, ao mesmo nível do `defaultLakehouse`. Os notebooks do repositório usam em vez dela um bloco `conf` com `spark.fabric.notebook.session.timeout`, que não consta dessa documentação. Num notebook novo, usar a chave documentada.

Três restrições da plataforma, todas documentadas pela Microsoft:

1. **Tem de ser a primeira célula de código.** Numa execução por pipeline, se não estiver na primeira célula a execução falha, porque a sessão já não pode ser reiniciada.
2. **Não é aplicada quando o notebook é invocado por `notebookutils.notebook.run`.** Continua a valer no `%run` e na execução interactiva.
3. **Mas é comparada.** Um notebook-filho só corre se usar o mesmo Lakehouse do pai, se herdar o do pai, ou se nenhum dos dois o definir. Se o filho indicar um Lakehouse **diferente**, a execução é bloqueada. A verificação contorna-se passando `useRootDefaultLakehouse: True` nos argumentos.

Daqui saem duas consequências práticas.

**Todos os notebooks de um pipeline devem declarar o mesmo Lakehouse de suporte.** Um `%%configure` divergente num notebook-filho não é ignorado em silêncio: bloqueia a execução quando o runner o invoca.

**Numa execução pelo runner, quem determina o `defaultLakehouse` é a sessão do runner**, e é de lá que vem o `/lakehouse/default/Files/Global_Libs` que o filho importa. A célula do filho continua a ser obrigatória, porque é ela que o torna executável isoladamente em desenvolvimento. Mas se o import das Global Libs falhar numa execução pelo runner, o `%%configure` a corrigir é o do **runner**.

Fontes: [Spark session configuration magic command](https://learn.microsoft.com/en-us/fabric/data-engineering/author-execute-notebook#spark-session-configuration-magic-command) e [NotebookUtils notebook run](https://learn.microsoft.com/en-us/fabric/data-engineering/notebookutils/notebookutils-notebook-run#considerations).

---

## Injecção de Parâmetros pelo Fabric

Os notebooks são chamados com:
```python
notebookutils.notebook.run(notebook_name, timeout_segundos, params)
```

A assinatura é `run(path: str, timeout_seconds: int = 90, arguments: dict = None, workspace: str = "")`. O `timeout_seconds` tem por omissão **90 segundos**, que é curto para um notebook de ingestão — daí os pipelines o passarem explicitamente. O quarto argumento é opcional e permite invocar um notebook noutro workspace.

Os parâmetros são serializados como JSON em três chaves:

| Chave | Conteúdo |
|---|---|
| `main_set` | configuração da execução, como string JSON |
| `running_local` | `False` quando chamado pelo runner |
| `log_correlation` | estado do ELogger serializado, para propagar o `correlation_id` |

**Porquê string e não dicionário:** os parâmetros de notebook do Fabric só suportam `int`, `float`, `bool` e `string` — `list` e `dict` não são suportados ([Assign parameters values from a pipeline](https://learn.microsoft.com/en-us/fabric/data-engineering/author-execute-notebook#assign-parameters-values-from-a-pipeline)). Daí o `json.dumps` do lado do runner, e o `json.loads` do lado do filho que aparece a seguir.

### A Cell de Parâmetros (tag "parameters")

Cada notebook filho tem uma cell marcada com a tag `parameters` nos metadados da cell (não no título — é uma propriedade da cell no Fabric). Esta cell contém os valores por defeito para execução local:

```python
# Esta cell tem a tag "parameters" nos metadados do Fabric
main_set = {
    "ws": "e_ext_ingest_ws",
    "pipeline_id": "nome_do_pipeline",
    # ... todos os parâmetros com valores por defeito para dev local
}
running_local = True
log_correlation = {}
```

**Como marcar uma cell como "parameters" no Fabric:** na interface do notebook, seleccionar a cell → menu "..." → "Toggle parameter cell". A cell fica marcada com um ícone de parâmetro.

### A Cell Injectada em Runtime

Quando o runner invoca o notebook via `notebookutils.notebook.run(...)`, o Fabric injeta automaticamente uma nova cell **imediatamente abaixo** da cell de parâmetros, em runtime. Esta cell não existe no ficheiro — é criada dinamicamente e contém os valores reais passados pelo runner:

```python
# Cell injectada pelo Fabric em runtime (não existe no ficheiro do notebook)
main_set = '{"ws": "e_ext_ingest_ws", "pipeline_id": "...", ...}'   # JSON string
running_local = False
log_correlation = '{"correlation_id": "20260414100000", ...}'
```

Os valores da cell injectada sobrepõem os da cell de parâmetros. Por isso, a cell seguinte em todos os notebooks faz sempre esta conversão:

```python
if type(main_set) == type(""):
    main_set = json.loads(main_set)
    log_correlation = json.loads(log_correlation)
```

Em execução local, `main_set` já é um dict e este bloco não faz nada. Em produção, chegou como JSON string e é convertido aqui.

### Tudo Parametrizável — Sem Hardcoding

Nenhum valor que varie por ambiente, por fonte, ou por pipeline pode estar hardcoded num notebook. Todos os valores operacionais chegam via `main_set`:

- Paths de Lakehouses (sempre ABFSS — ver [Decisões Arquitecturais](visao-geral.md#decisões-arquitecturais))
- Nomes de contentores Azure
- Credenciais (nunca no código — sempre via Key Vault)
- Nomes de tabelas Delta
- Pastas de Bronze/Silver/Gold

Isto garante que o mesmo notebook corre em qualquer ambiente apenas mudando os parâmetros no runner, sem tocar no código.

---

## Execução Local — Correr um Notebook Isolado

Para testar ou re-processar um notebook sem passar pelo runner:

### Passo 1 — Abrir o notebook no Fabric

Abrir directamente o notebook que se quer correr (ex: `<dominio>_silver_fact_<fonte>`).

### Passo 2 — Configurar a cell de parâmetros

A cell marcada com a tag `parameters` contém os valores por defeito para execução local. Editar directamente essa cell com os valores pretendidos:

```python
main_set = {
    "ws"            : "e_ext_ingest_ws",
    "pipeline_id"   : "<dominio>",
    "lh_origem"     : "abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>/",
    "lh_destino"    : "abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>/",
    "bronze_folder" : "Files/<dominio>/ingest/<fonte>",
    "mask_ficheiro" : "*.csv",
    "schema_table"  : "<dominio>.silver_<fonte>",
    # ... restantes parâmetros específicos do notebook
}
running_local = True   # ← obrigatório para execução local
log_correlation = {}
```

**Importante:** `running_local = True` faz com que erros não terminem o notebook imediatamente (levantam excepção em vez de `notebookutils.notebook.exit()`), o que permite ver o traceback completo.

### Passo 3 — Correr as células de imports e bibliotecas primeiro

Os notebooks dependem das bibliotecas das Global Libs. Correr as primeiras células (imports e `sys.path.append`) antes das células de lógica:

```python
import sys
sys.path.append("/lakehouse/default/Files/Global_Libs")
from dap_ELogger import ELogger
from dap_lake_file_tools import list_files_by_date_mask_startingDate
# etc.
```

Este import depende do Lakehouse de **suporte** estar declarado como `defaultLakehouse` na célula `%%configure -f` do notebook — é lá que vive o `Files/Global_Libs`. Não é o Lakehouse de dados do pipeline, que é alcançado só por ABFSS a partir do `main_set`. Se o import falhar, corrige-se o `%%configure`; as libs nunca se copiam para o Lakehouse de dados.

### Passo 4 — Correr o notebook sequencialmente

Depois de configurar a cell de parâmetros e correr os imports, correr o resto do notebook célula a célula (ou "Run all" se os parâmetros estiverem correctos).

### Notas sobre notebooks Gold com steps configuráveis

Os notebooks Gold têm uma lista de steps que controla que tabelas são construídas nessa execução. Onde ela vive depende da geração do runner:

| Geração | Onde | Forma |
|---|---|---|
| Actual | `main_set["task_config"]["running_steps"]` | lista de nomes de step |
| Antiga | `main_set["running_steps"]` | lista de dicts `{"id": ..., "enabled": ...}` |

Para correr um step isolado, editar essa lista na cell de parâmetros. Ver [Camada Gold](camada-gold.md#steps-configuráveis).

---

[← Camada Gold](camada-gold.md) · [Bibliotecas →](bibliotecas-comuns.md) · [Home](../Home.md)
