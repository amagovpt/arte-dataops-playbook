# Bibliotecas Comuns

As bibliotecas partilhadas estão em `/lakehouse/default/Files/Global_Libs/` e são compiladas por notebooks dedicados (`common_nb_compile_*`).

O Lakehouse de suporte, onde vivem as Global Libs, é declarado como `defaultLakehouse` na célula `%%configure -f` de cada notebook. Isso permite o import directo pelo path `/lakehouse/default/`:

```python
import sys
sys.path.append("/lakehouse/default/Files/Global_Libs")
from dap_ELogger import ELogger
```

Este path funciona porque o Lakehouse de **suporte** é declarado como `defaultLakehouse` na célula `%%configure -f` do notebook — é o mecanismo, e não depende de ninguém afixar nada pela UI do Fabric. O Lakehouse de suporte não é o Lakehouse de dados do pipeline: as Global Libs vivem só no de suporte e nunca se copiam para o de dados. Este é o **único** `/lakehouse/default/` legítimo — para tudo o resto (dados, tabelas Delta), usar sempre ABFSS explícito derivado do `main_set`, ver [Decisões Arquitecturais](visao-geral.md#decisões-arquitecturais).

---

## Como se Publica uma Biblioteca

Cada `common_nb_compile_<lib>.ipynb` tem o código da biblioteca dentro de uma string e escreve-o
como ficheiro:

```python
file_location = "/lakehouse/default/Files/Global_Libs/dap_<lib>.py"
with open(file_location, "w") as ficheiro:
    ficheiro.write(code)
```

Publicar é correr esse notebook com o Lakehouse de suporte declarado no `%%configure`. Não há
build, nem empacotamento, nem versão. Três consequências a ter presentes:

1. **Não há versionamento.** Alterar uma biblioteca afecta todos os pipelines que a importam, na
   execução seguinte, sem aviso. Não há forma de um pipeline ficar preso a uma versão anterior.
   Uma alteração incompatível parte tudo ao mesmo tempo.
2. **O que está publicado não é necessariamente o que se lê no notebook.** Os notebooks de
   compilação têm células de teste e rascunhos ao lado da célula que escreve o ficheiro, por vezes
   com versões diferentes da mesma função. Só a célula que faz `write` conta. Ao investigar o
   comportamento de uma lib, confirmar qual é a célula que escreve.
3. **A sessão importa a biblioteca uma vez.** Depois de republicar, é preciso reiniciar a sessão
   dos notebooks que a usam, senão continuam com a versão antiga em memória.

### Porquê Global Libs e não um Environment do Fabric

O Fabric tem Environments, que é o mecanismo próprio para distribuir bibliotecas. Não são usados,
por três razões:

1. As dependências de que as `dap_*` precisam já vêm no runtime. Não há nada a instalar.
2. Um Environment custa minutos na publicação e acrescenta tempo ao arranque de cada sessão,
   contra segundos do starter pool.
3. Passaria a haver **dois** mecanismos de distribuição de código a conviver.

Environment passa a fazer sentido no dia em que for preciso um pacote que não venha no runtime,
ou em que se queira empacotar as `dap_*` num `.whl` versionado — o que resolveria o ponto 1 da
secção anterior. Nessa altura é uma migração de todas as bibliotecas, não um Environment avulso.

### `mssparkutils` e `notebookutils`

As bibliotecas usam `mssparkutils`. A Microsoft renomeou esse pacote para `notebookutils`: o nome
antigo continua a funcionar por compatibilidade, mas as funcionalidades novas só saem no
`notebookutils` e o `mssparkutils` será retirado
([NotebookUtils, former MSSparkUtils](https://learn.microsoft.com/en-us/fabric/data-engineering/notebook-utilities)).
Código novo deve usar `notebookutils`.

---

## `dap_ELogger.py`

**Compilado por:** `common_nb_compile_ELogger.ipynb`  
**Ficheiro em produção:** `/lakehouse/default/Files/Global_Libs/dap_ELogger.py`

Sistema de logging estruturado que escreve eventos em formato **Delta Lake** no Lakehouse de observabilidade partilhado. Cada evento é uma linha Delta — consultável via SQL, com partição por dia.

---

### Onde ficam os logs

```
Workspace:  e_common_ws
Lakehouse:  common_lh_observability
Tabela:     Tables/ELogger/pipeline_logs
Caminho:    abfss://e_common_ws@onelake.dfs.fabric.microsoft.com/
            common_lh_observability.Lakehouse/Tables/ELogger/pipeline_logs
Partição:   log_date  (derivada do campo timestamp, UTC)
```

Para consultar via SQL no Fabric:
```sql
SELECT * FROM common_lh_observability.ELogger.pipeline_logs
WHERE pipeline_id = 'meu_pipeline'
ORDER BY timestamp DESC
LIMIT 100
```

---

### Schema da Tabela de Logs

| Coluna | Tipo | Descrição |
|---|---|---|
| `timestamp` | `TimestampType` | Momento do evento (UTC) |
| `workspace` | `StringType` | Workspace onde correu |
| `pipeline_id` | `StringType` | Identificador do pipeline (ex: `meu_pipeline`) |
| `notebook` | `StringType` | Nome do notebook que gerou o evento |
| `step_name` | `StringType` | Passo dentro do notebook (ex: `"10 - blobs encontrados"`) |
| `group` | `StringType` | Grupo de execução (ex: `carregamento-ccdr-centro`) |
| `level` | `StringType` | Tipo do evento: `info`, `warn`, `error`, `dump`, `rowCount` |
| `message` | `StringType` | Mensagem livre |
| `correlation_id` | `StringType` | ID partilhado em toda a execução do pipeline |
| `extra` | `StringType` | Dados adicionais em formato livre (JSON, texto) |
| `s_from_start` | `IntegerType` | Segundos decorridos desde a criação da instância ELogger |
| `step_no` | `IntegerType` | Contador de eventos, incrementado a cada chamada e propagado para os notebooks-filho |
| `log_date` | `DateType` | Coluna de partição, derivada de `timestamp` |

---

### Como Funciona — Fluxo Completo

#### 1. Instanciar no runner (início do pipeline)

```python
elogger = ELogger(
    workspace   = ws,           # "e_ext_ingestions_ws"
    pipeline_id = pipeline_id,  # "meu_pipeline"
    group       = group,        # "main"
    nb          = nb,           # "meu_pipeline_runner"
    logLevel    = 5,            # verbosidade máxima (ver secção logLevel)
    printOnScreen = True        # imprime também no output do notebook
)
elogger.log_event_start(message="group:main")
```

Neste momento é gerado automaticamente um `correlation_id` no formato `yyyyMMddHHmmss`. Este ID vai identificar esta execução completa do pipeline em todos os logs.

#### 2. Propagar para notebooks filhos

Quando o runner invoca um notebook filho, passa o estado do logger serializado:

```python
call_parameters = {
    "main_set"        : json.dumps(nb_params),
    "running_local"   : False,
    "log_correlation" : json.dumps(elogger.get_correlation())
}
notebookutils.notebook.run(notebook_name, timeout, call_parameters)
```

As três chaves são obrigatórias. Sem `running_local: False`, o filho fica com o valor por defeito
da sua célula `parameters` e corre como se fosse execução manual: em caso de erro levanta excepção
em vez de devolver ao runner o JSON com o `status` e o `errorId`.

`get_correlation()` devolve um dicionário com todo o estado interno: `ts_initial`, `correlation_id`, `pipeline_id`, `logLevel`, `printOnScreen`, `commulativeWriteTime`, `lastLogTime`, etc.

#### 3. Reconstruir no notebook filho

```python
# log_correlation chega do runner como string JSON, e o construtor precisa de um dict
if type(log_correlation) == type(""):
    log_correlation = json.loads(log_correlation)

elogger = ELogger(
    correlation = log_correlation,  # estado do runner, já desserializado
    workspace   = ws,
    nb          = nb                # nome deste notebook filho
)
```

Nos notebooks, esta desserialização acontece na mesma linha que a do `main_set`, logo a seguir à
célula de parâmetros ([ver Orquestração](orquestracao-runner.md#a-cell-injectada-em-runtime)). Se o
`log_correlation` chegar ao construtor ainda como string, o acesso às chaves do estado falha.

O notebook filho fica com o **mesmo `correlation_id`** do runner. Todos os eventos gravados por ele aparecem na mesma "sessão" na tabela de logs.

#### 4. Gravar eventos ao longo do notebook

```python
elogger.log_event_info(
    step_name = "10 - blobs encontrados",
    message   = f"blobs encontrados: {len(blobs)}",
    logLevel  = 5,
    extra     = ""
)

elogger.log_event_error(
    step_name = "error nb",
    message   = json.dumps(errorState),
    logLevel  = 1
)
```

#### 5. Terminar

```python
elogger.log_event_Finish(message="group:main")
```

---

### Níveis de Log (`logLevel`)

Existem dois conceitos distintos com o mesmo nome que é importante não confundir:

- **`level`** — o *tipo* do evento: `info`, `warn`, `error`, `dump`, `rowCount`. É gravado na coluna `level` da tabela Delta e serve para filtrar ao consultar os logs.
- **`logLevel`** — um número que controla a *verbosidade*. Define o que é efectivamente escrito na tabela Delta.

#### Como funciona o threshold

O ELogger tem um `logLevel` de instância (definido na criação). Cada chamada de log tem também um `logLevel` de chamada. **O evento só é gravado se `logLevel_chamada ≤ logLevel_instância`.**

| `logLevel` instância | Eventos que passam o threshold |
|---|---|
| `1` | Os que são chamados com `logLevel = 1`: o start, o finish, o `log_rowCount`, e qualquer chamada onde se tenha passado `logLevel=1` explicitamente |
| `2` | Os anteriores, mais tudo o que use o valor por defeito dos helpers (`2`) — suficiente para produção normal |
| `5` | Tudo, incluindo os eventos de debug marcados com `logLevel=5` — usado em desenvolvimento e actualmente no runner em produção |

Passar o threshold não garante que o evento fique gravado: a escrita pode falhar, e nesse caso o
ELogger imprime o erro e continua ([ver Resiliência](#resiliência)). A ausência de uma linha na
tabela não prova a ausência do evento.

#### Métodos disponíveis e os seus defaults

| Método | `level` gravado na tabela | `logLevel` de chamada por defeito |
|---|---|---|
| `log_event_start(...)` | `info` | 1 |
| `log_event_Finish(...)` | `info` | 1 |
| `log_event_info(step, msg)` | `info` | 2 |
| `log_event_warn(step, msg)` | `warn` | 2 |
| `log_event_error(step, msg)` | `error` | 2 |
| `log_event_dump(step, msg)` | `dump` | 2 |
| `log_rowCount(step, rowCount)` | `rowCount` | 1 |

O `log_rowCount` grava a contagem na coluna `message`, com `level = "rowCount"`. É o que permite
consultar volumes por execução sem parsear mensagens livres. Não confundir o `level` com a lista
de níveis de severidade: `rowCount` é um quinto valor possível na coluna.

#### Ajustar o `logLevel` por chamada

O `logLevel` pode ser sobreposto em cada chamada individual. Exemplos práticos:

```python
# Grava mesmo que a instância esteja em logLevel=1 (crítico — não ignorar)
elogger.log_event_error(step_name="error nb", message=json.dumps(errorState), logLevel=1)

# Só grava se a instância estiver em logLevel=5 (detalhe de debug)
elogger.log_event_info(step_name="blobs listados", message=f"{len(blobs)} blobs", logLevel=5)

# Comportamento padrão — grava em logLevel >= 2
elogger.log_event_info(step_name="ficheiro processado", message=nome_ficheiro)
```

#### Consultar por severidade

```sql
-- Ver apenas erros de uma execução
SELECT * FROM common_lh_observability.ELogger.pipeline_logs
WHERE pipeline_id = 'meu_pipeline'
  AND level = 'error'
ORDER BY timestamp DESC

-- Ver o resumo de uma execução (só start/finish)
SELECT notebook, step_name, message, s_from_start
FROM common_lh_observability.ELogger.pipeline_logs
WHERE correlation_id = '20260414100000'
  AND step_name IN ('Start', 'Finish')
ORDER BY timestamp
```

---

### `printOnScreen`

Se `printOnScreen=True` na instância (ou `printOnScreen_thisMsf=True` na chamada), cada evento é também impresso no output do notebook com o formato:

```
start:7s   last:1s ### Log:[blobs encontrados: 12]
```

- `start` — segundos desde a criação do logger
- `last` — segundos desde o log anterior

Útil para acompanhamento em tempo real durante desenvolvimento (`running_local = True`).

---

### Resiliência

O método `log_event` captura todas as excepções internamente. Se a escrita Delta falhar (ex: problemas de rede, permissões), o erro é impresso no ecrã mas **o pipeline não é interrompido**:

```python
except Exception as e:
    print("[ELogger][ERROR] Failed to write log to Delta.")
    print(f"  Original message: {message}")
    print(f"  Pipeline/step: {pipeline_id}.{step_name}")
    print(f"  Error: {repr(e)}")
```

---

### Custo — nunca logar dentro de um ciclo

Cada chamada que grava faz um `spark.createDataFrame` e um commit Delta na tabela de logs, que
está noutro workspace. É um job Spark e uma transacção por linha registada. O custo é **fixo por
evento**: uma mensagem de uma linha custa o mesmo que um dump.

**O que se paga é o Spark, não o volume.** Gravar uma linha custa o mesmo que gravar um milhão:
o custo é o arranque do job e o commit, não os dados. Num incidente registado, um ciclo de 308
iterações, com cerca de três logs cada — executados mesmo nas iterações que não tinham trabalho nenhum —
gerou 928 escritas e 2h11m, contra 11 segundos de trabalho útil. Cerca de 8 segundos por chamada.

Nota adicional sobre a tabela de logs: um `append` por linha deixa um ficheiro Parquet minúsculo
por evento registado, e não há compactação configurada.

O padrão a seguir:

1. Acumular os resultados numa lista, dentro do ciclo.
2. Logar **uma vez**, no fim, com o resumo.
3. Dentro do ciclo, logar só por excepção — quando a iteração faz mesmo alguma coisa fora do
   comum, ou falha.

Um evento cujo `logLevel` de chamada seja superior ao da instância não é gravado e não custa
nada. Baixar o `logLevel` mitiga, mas não substitui tirar o log de dentro do ciclo.

O ELogger acumula internamente o tempo total gasto a escrever, em segundos, acessível por
`elogger.get_seconds()` — serve para medir exactamente este overhead numa execução.

---

## `arteutils.py`

**Compilado por:** `common_nb_utils.ipynb`  
**Ficheiro em produção:** `/lakehouse/default/Files/Global_Libs/arteutils.py`

Classe utilitária mínima. Actualmente contém apenas o registo do timestamp de início:

```python
class arteutils:
    def __init__(self, app_name):
        self.ts_initial = datetime.now()
```

Não é utilizada pelos notebooks de pipeline actualmente.

---

## `dap_json_validator.py`

**Compilado por:** `common_nb_compile_json_validator.ipynb`

Valida payloads JSON contra o [Catálogo Nacional de Modelos de Dados](https://github.com/amagovpt/CATALOGO_MODELOS_DADOS_TI), **offline** — nunca vai à rede. Núcleo derivado do validador do ISEL (© Nuno Datia, FIT/ISEL, licença MIT).

Depende de `jsonschema` e `referencing`, ambos já presentes no Fabric Runtime 1.3 — não é preciso Environment. A verificação de `format` (ex.: `date-time`) só está activa porque o runtime traz também `rfc3339-validator` e `rfc3986-validator`; sem essas bibliotecas o `jsonschema` ignora os formatos **em silêncio**.

### Função principal

```python
def validate_payload(payload, bundle_path, modelo=None, ngsi_ld=True, spark=None) -> dict
```

É a única função que um notebook Bronze precisa de chamar. Trata sozinha de carregar o bundle (uma vez por sessão), detectar o modelo e resolver os `$ref`. Não levanta excepção por causa do conteúdo do payload.

| Parâmetro | Descrição |
|---|---|
| `payload` | O registo a validar (dict), ou uma lista de registos com o mesmo `type` |
| `bundle_path` | Path ABFSS do `schemas_bundle.json`, vindo do `main_set` |
| `modelo` | Opcional — força um modelo. Por defeito é detectado do próprio payload |
| `ngsi_ld` | Converte NGSI-LD normalizado para key-values antes de validar. **Manter `True`** — sem a conversão a validação dá lixo |
| `spark` | Sessão Spark, necessária para ler o bundle por ABFSS |

Devolve um dict pronto a virar linha de tabela:

```python
{"estado": "NAO_CONFORME", "modelo": "WeatherObserved", "n_erros": 1,
 "erros": [{"campo": "$.dateObserved", "regra": "properties/dateObserved/format",
            "tipo": "format", "mensagem": "'2024-10-20T04:00:00.00' is not a 'date-time'"}]}
```

### Os cinco estados

| Estado | Significado | `erros` |
|---|---|---|
| `CONFORME` | Validou sem erros | vazio |
| `NAO_CONFORME` | Falhou em campos concretos | preenchido |
| `MODELO_DESCONHECIDO` | `type` fora do catálogo, ou lista com modelos diferentes | vazio |
| `NAO_E_DCAT` | Sem `type` nem `@type`, lista vazia ou com não-objectos | vazio |
| `GRAPH_NAO_PERMITIDO` | `@graph` no payload ou em qualquer elemento da lista | vazio |

**Três dos cinco devolvem `erros` vazio.** Quem escrever numa tabela de erros tem de guardar o estado à parte, senão a razão da rejeição desaparece.

Um sexto estado, `JSON_INVALIDO`, não vem da lib — é do notebook, quando o `json.loads` falha antes de a lib ser chamada.

### Funções auxiliares

| Função | Para quê |
|---|---|
| `load_bundle(bundle_path, spark=None)` | Lê o bundle e monta o `Registry`. Chamada automaticamente pela `validate_payload`, com cache por sessão |
| `detectar_modelo(payload, bundle)` | Devolve `(modelo, estado)`. Dados pelo campo `type`, metadados pelo `@type` DCAT |
| `ngsi_ld_to_key_values(instance)` | Conversão NGSI-LD normalizado → key-values |
| `relatorio_markdown(resultado, titulo=None)` | Relatório legível de **um** payload, para depuração pontual |

### Notas de utilização

- **O `@graph` é detectado em código, não pelo schema.** Os schemas do catálogo têm `additionalProperties: true`, portanto um `@graph` ao lado de campos válidos passa com zero erros. A proibição foi determinada pelo negócio e está escrita na `detectar_modelo`. Não há flag para a desligar.
- **A variante INSPIRE é detectada pela presença de campos `ins:`.** `Metadata.Dataset` e `Metadata.Dataset.inspire` são iguais tirando esses quatro campos, portanto um dataset que devia ser INSPIRE mas vem sem nenhum deles valida como `Metadata.Dataset` e dá conforme. Para forçar, passar `modelo="Metadata.Dataset.inspire"`.
- **Aceita ficheiros em array**, desde que todas as entidades tenham o mesmo `type`. No Bronze prefere-se iterar: com a lista inteira os erros vêm por posição (`$[0]`) e não pelo `id`.
- **O bundle é um instantâneo do catálogo**, com os 90 modelos e os documentos externos que eles referenciam, num único ficheiro. Uma leitura em vez de mais de noventa, e um estado coerente. Refrescá-lo é passo deliberado — nenhum pipeline o reconstrói.
- **Custo:** cerca de 0,5 ms por validação. O caro é por sessão (ler o bundle e montar o `Registry`), não por registo. Corre no driver, single-thread — não escala para validação linha a linha de milhões de registos sem outra arquitectura.
- **Acumular fora do ciclo.** Cada `elogger.log_event_*` custa cerca de 8 s; a validação custa 0,5 ms. Acumular os resultados e logar uma vez no fim.

---

## `dap_key_vault.py`

**Compilado por:** `common_nb_compile_key_vault.ipynb`

Lê segredos do Azure Key Vault via `mssparkutils.credentials.getSecret`.

### Função

```python
def read_kv_secret(vault_or_conn: str, secret_name: str, vault_name: str | None = None) -> str
```

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `vault_or_conn` | URL do Key Vault ou nome de ligação | `"https://<nome-do-vault>.vault.azure.net/"` |
| `secret_name` | Nome do segredo | `"<pipeline>-<fonte>-<tipo>"` |
| `vault_name` | Opcional — nome do vault (para assinatura alternativa) | — |

A função tenta dois formatos de chamada ao `mssparkutils` por compatibilidade. Lança `RuntimeError` se nenhum funcionar.

### Regras de Segurança — Passwords e Credenciais

- **Nunca hardcode** uma password, SAS token, connection string ou API key num notebook — sempre via Key Vault
- **Nunca logar** o valor de um segredo (não passar o resultado de `read_kv_secret` para o ELogger)
- O URL do Key Vault e o nome do segredo chegam sempre via `main_set` — são parâmetros, não constantes
- Adicionar novos segredos directamente no Azure Key Vault (URL definido por ambiente) e referenciar o nome no `main_set` do runner

### Como Adicionar um Novo Segredo

1. Adicionar o segredo no Azure Key Vault com um nome descritivo (ex: `<pipeline>-<fonte>-<tipo>`)
2. No runner, no `datasources` ou na task, referenciar o nome do segredo:
   ```python
   "secret_to_read": "nome-do-segredo-no-keyvault"
   ```
3. No notebook que precisa do valor, ler via:
   ```python
   secret = read_kv_secret(main_set["key_vault"], main_set["secret_to_read"])
   ```

O segredo nunca aparece no código — só o seu nome.

---

## `dap_azure_blob.py`

**Compilado por:** `common_nb_compile_azure_blob.ipynb`

Funções REST para interagir com Azure Blob Storage (sem SDK Azure, apenas `requests`).

### `list_blobs_via_rest`

```python
def list_blobs_via_rest(account_name: str, container_name: str, sas_token: str) -> list[dict]
```

Chama a REST API (`?restype=container&comp=list`) e faz parse do XML de resposta. Devolve uma lista de dicionários com os metadados de cada blob, incluindo `name`, `properties_content_length`, `properties_blobtype`, `properties_last_modified`, entre outros.

### `read_blob_via_rest`

```python
def read_blob_via_rest(account_name: str, container_name: str, file_name: str, sas_token: str) -> requests.Response
```

Faz GET do blob e devolve o objecto `Response` (para streaming com `iter_content`). Lança excepção se o status HTTP não for 2xx.

### `delete_blob_via_rest`

```python
def delete_blob_via_rest(account_name: str, container_name: str, blob_name: str, sas_token: str) -> str
```

Faz DELETE do blob. Aceita status `202` ou `204` como sucesso. Lança `RuntimeError` em caso de falha.

---

## `dap_lake_file_tools.py`

**Compilado por:** `common_nb_compile_lake_file_tools.ipynb`

Utilitários para manipulação de ficheiros no Lakehouse.

### `upload_from_requestes_file_in_chunks`

```python
def upload_from_requestes_file_in_chunks(resp, lh_destino: str, folder_destino: str, file_name: str)
```

Escreve um ficheiro no Lakehouse em chunks de **4 MB** usando a Azure Data Lake Storage Gen2 REST API. Ver [Camada Bronze](camada-bronze.md) para detalhe do protocolo.

### `extract_date_from_name`

```python
def extract_date_from_name(filename: str, DATE_PATTERNS: list = None) -> Optional[datetime]
```

Extrai a data do início do nome de um ficheiro. Padrões suportados por defeito:

| Padrão regex | Formato |
|---|---|
| `^(\d{12})_` | `yyyyMMddHHmm` |
| `^(\d{8})_` | `yyyyMMdd` |

Devolve `None` se não houver correspondência.

### `list_files_by_date_mask_startingDate`

```python
def list_files_by_date_mask_startingDate(path: str, suffix_filter: str) -> tuple[list, list]
```

Lista ficheiros num directório do Lakehouse (via `mssparkutils.fs.ls`), filtra por `suffix_filter` (glob, ex: `*_HabSimplificado.csv`), extrai datas dos nomes e ordena do mais recente para o mais antigo.

**Devolve:**
- `matched` — lista de dicts `{name, path, size_bytes, file_date, modified_ts}`, ordenada por `file_date` descendente
- `skipped` — lista de nomes de ficheiros que passaram no filtro glob mas cujo nome não tem data válida

**O nome engana: não há filtro por data inicial.** Apesar do `startingDate`, a função não recebe nem aplica data de corte — devolve todos os ficheiros que passam no glob, ordenados por `file_date` descendente. Qualquer corte por data de arranque é feito **depois**, sobre o `matched`, ou implica alterar a função.

Os notebooks silver usam `matched[0]` para seleccionar o ficheiro mais recente quando `nome_ficheiro` não é especificado.

### `upload_jsonl_to_abfss_in_chunks`

```python
def upload_jsonl_to_abfss_in_chunks(data_list, lh_destino: str, folder_destino: str, file_name: str) -> str
```

Serializa uma lista de dicts para JSONL e escreve-a no Lakehouse pelo mesmo protocolo de chunks de 4 MB da `upload_from_requestes_file_in_chunks`, acumulando linhas até atingir o tamanho do chunk. Devolve o path ABFSS do ficheiro escrito. Usar quando o que se tem é uma lista em memória e não uma resposta HTTP.

### `format_filename`

```python
def format_filename(file_name: str, reference_date, include_time: bool = True) -> str
```

Prefixa o nome com uma data: `yyyyMMdd_HHmmss_<nome>` ou `yyyyMMdd_<nome>`. É a contraparte da `extract_date_from_name` — quem escreve com esta consegue ler com aquela.

### `get_files_by_mask`

```python
def get_files_by_mask(directory_path: str, mask: str = "*projetos*.jsonl") -> list[str]
```

Lista os paths completos dos ficheiros de um directório que correspondem a uma máscara glob. Devolve lista vazia e imprime o erro se o directório não for acessível — não levanta excepção. Ao contrário da `list_files_by_date_mask_startingDate`, não extrai datas nem ordena.

### `list_abfss_files_by_pattern`

```python
def list_abfss_files_by_pattern(path: str, pattern: str) -> list[str]
```

Lista ficheiros cujo nome contém `pattern` (comparação em minúsculas, sem glob). Não é recursiva.

**Não usar.** A função chama `item.isDir()` como método, mas `isDir` é uma propriedade booleana
([notebookutils.fs.ls](https://learn.microsoft.com/en-us/fabric/data-engineering/notebookutils/notebookutils-file-system#view-file-properties))
— e é assim que as restantes funções desta biblioteca a usam. A chamada levanta `TypeError`, que
o `except` da própria função apanha: imprime o erro e devolve **lista vazia**, sem falhar. Nenhum
pipeline a usa. Usar a `get_files_by_mask`.

### `delete_abfss_files`

```python
def delete_abfss_files(file_paths, recursive: bool = False) -> int
```

Apaga um path ou uma lista de paths. Devolve quantos foram apagados. Não levanta excepção: um ficheiro que não exista imprime aviso e conta como não apagado. Confirmar sempre o valor devolvido contra o número de ficheiros que se esperava apagar.

---

## `dap_fabric_config_tools.py`

**Compilado por:** `common_nb_compile_fabric_config_tools.ipynb`  
**Ficheiro em produção:** `/lakehouse/default/Files/Global_Libs/dap_fabric_config_tools.py`

Utilitários de orquestração usados pelo runner e pelos notebooks filhos.

### `handleError`

```python
def handleError(step_number, e, running_local, results=[])
```

Tratamento de erros com saída imediata. Imprime o erro, monta um dicionário com `status`,
`errorState`, `errorId` e `previous_results`, e interrompe:

| `running_local` | O que faz |
|---|---|
| `False` (Fabric, chamado pelo runner) | `notebookutils.notebook.exit(json.dumps(result))` |
| `True` (execução manual) | levanta `KeyError` com o mesmo JSON, para se ver o traceback |

O dicionário leva **sempre** também `"files_copied": 0`, sobra de quando a função só servia o
Bronze. Uma Silver ou uma Gold que falhe devolve esse campo na mesma, e ele não significa nada
fora do Bronze. O runner não o lê — lê só o `new_data`, que o `handleError` não põe.

**Nunca devolve.** Não há valor de retorno a atribuir — `errorState, errorId = handleError(...)`
é a assinatura de uma versão antiga, definida dentro dos notebooks, e não funciona com a da
biblioteca. O quarto argumento, `results`, é o que já se tinha feito até ali, para não se perder
no JSON devolvido ao runner.

**Onde este padrão se parte.** O `notebook.exit()` termina o notebook **lançando uma excepção**.
Daí o aviso da Microsoft para não o chamar dentro do corpo de um `try`: o `except` desse bloco
apanha a excepção do próprio `exit` e o notebook continua a correr como se nada fosse
([NotebookUtils notebook run](https://learn.microsoft.com/en-us/fabric/data-engineering/notebookutils/notebookutils-notebook-run#exit-a-notebook)).

O `handleError` é chamado de dentro do `except`, não do `try`, e uma excepção levantada dentro de
um handler propaga-se para fora — por isso o padrão normal não cai nesta armadilha. Cai-se nela
ao embrulhar a chamada a `handleError`, ou ao próprio `exit`, num segundo `try` com um `except`
largo. Não fazer isso.

Ver [Camada Silver — Tratamento de Erros](camada-silver.md#tratamento-de-erros).

### `read_config_pipelines`

```python
def read_config_pipelines(file_json) -> dict
```

Lê a configuração de um pipeline a partir de um ficheiro JSON no Lakehouse. O path é fixo, e o
workspace e o Lakehouse estão **escritos no código da função**:

```
abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>/Files/configs/pipelines/{file_json}.json
```

Recebe só o nome do ficheiro, sem extensão. Devolve o dict de configuração. Copia o ficheiro
para disco local com `mssparkutils.fs.cp` e lê-o inteiro — **não há limite de tamanho**. Uma
versão anterior usava `mssparkutils.fs.head`, cujo tecto por defeito é 100 KB, e truncava a
configuração em silêncio a partir daí. Se encontrar um erro de JSON inválido numa configuração
grande, é esse o primeiro sítio a verificar.

Mudar de Lakehouse de configuração implica recompilar a biblioteca — ver
[Orquestração → Onde vive o ficheiro](orquestracao-runner.md#onde-vive-o-ficheiro).

### `build_call_nb_params`

```python
def build_call_nb_params(config: dict, group_order: int, task_order: int) -> dict
```

Constrói o dicionário de parâmetros passado a cada notebook filho. Faz merge de: configuração global (`allTasks`), informação do pipeline (`pipeline_info`), contexto do grupo e configuração da task específica. Devolve dicionário vazio se o grupo ou task não forem encontrados.

### `update_fabric_metadata`

```python
def update_fabric_metadata(config, key=None, new_value=None, group_order=None, task_order=None)
```

Actualiza o config em runtime — a nível raiz, de grupo ou de task, consoante os parâmetros fornecidos. Permite que o runner actualize estado (ex: marcar uma task como concluída) sem reescrever o ficheiro de configuração.

### `run_notebook_with_params`

```python
def run_notebook_with_params(toRun_notebook: str, timeout: int, params: dict)
```

Invoca `notebookutils.notebook.run()` e devolve o resultado JSON já desserializado.

**No caminho de erro não devolve nada.** Monta um dicionário com `status: "ERROR"` e `errorId: -1`
e levanta `KeyError` com esse dicionário serializado — o `return` nunca chega a ser executado. Para
ler o erro é preciso apanhar o `KeyError` e desserializar `e.args[0]`.

### `get_execution_plan`

```python
def get_execution_plan(main_set) -> list[dict]
```

Lê os grupos e tasks do `main_set`, ordena por `order` (grupos e tasks), e devolve uma lista plana de dicts com o plano de execução completo — útil para logging e para iterar sobre as tasks sem gerir a hierarquia manualmente.

---

## `dap_fabric_semantic_tools.py`

**Compilado por:** `common_nb_compile_fabric_sm_tools.ipynb`  
**Ficheiro em produção:** `/lakehouse/default/Files/Global_Libs/dap_fabric_semantic_tools.py`

Utilitários para refresh de modelos semânticos Power BI (DirectLake). Assentam em `sempy.fabric`, que vem no runtime — não é preciso instalar nada.

### `semantic_model_refresh`

```python
def semantic_model_refresh(dataset_name, workspace_id) -> str | None
```

Faz trigger de refresh via `fabric.refresh_dataset()`. Devolve o `request_id` do refresh ou `None` em caso de falha.

### `semantic_model_refresh_request_history`

```python
def semantic_model_refresh_request_history(dataset_name, workspace_id, rquest_id)
```

Consulta o estado de um refresh em curso via `fabric.get_refresh_execution_details()`. Devolve um objecto com os atributos `.status`, `.start_time` e `.end_time`.

### `semantic_model_refresh_wait`

```python
def semantic_model_refresh_wait(dataset_name, workspace_id, wait_s=5, times=5)
```

Faz trigger do refresh e aguarda a conclusão em polling. Devolve o objecto de histórico do último
`status` lido.

| Comportamento | Detalhe |
|---|---|
| Intervalo entre verificações | `wait_s` segundos, por defeito 5 |
| Máximo de tentativas | `times`, por defeito 5 |
| Se não concluir a tempo | Lança `Exception("Refresh não concluido: {status}")` |
| Ao concluir | Imprime a duração total em segundos |

Os defaults dão **25 segundos** de espera total, curto para qualquer modelo real. Passar `wait_s` e `times` explicitamente.

Usado no final da camada Gold e no runner como step `sm_refresh`, quando o pipeline tem modelo semântico associado.

---

[← Orquestração](orquestracao-runner.md) · [Fontes de Dados →](fontes-de-dados.md) · [Home](../Home.md)
