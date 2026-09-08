# Fontes de Dados

Padrões de acesso a fontes externas usados nos pipelines.

---

## Azure Blob Storage

Acesso via **REST API** com **SAS Token** lido do Azure Key Vault. Não é usado nenhum SDK Azure — apenas `requests`.

```python
# Ler SAS Token do Key Vault
sas_token = read_kv_secret(key_vault_url, secret_name)

# Listar blobs
blobs = list_blobs_via_rest(account_name, container_name, sas_token)

# Descarregar um blob (streaming)
response = read_blob_via_rest(account_name, container_name, file_name, sas_token)

# Apagar blob
delete_blob_via_rest(account_name, container_name, file_name, sas_token)
```

Implementação em `dap_azure_blob.py` — ver [Bibliotecas Comuns](bibliotecas-comuns.md#dap_azure_blobpy).

### Comportamento Consume-and-Delete

O blob é apagado da origem **após** confirmação do upload para o Lakehouse. Se o upload falhar, o blob permanece disponível para reprocessamento.

---

## REST API Pública

Para APIs que expõem dados em CSV ou JSON sem autenticação. Padrão típico: descobrir o catálogo
de recursos → iterar sobre cada um → GET e parse → escrever em Delta.

Quatro coisas que um Bronze sobre API pública precisa de ter, e que uma chamada `requests.get`
simples não tem:

| | Porquê |
|---|---|
| **Retry com backoff crescente** | Um 5xx transitório não pode falhar o pipeline. Repetir com espera crescente, e falhar com uma mensagem que diga o URL e quantas tentativas foram feitas |
| **Sessão `requests` reutilizada** | Evita renegociar a ligação a cada pedido. Numa extracção de centenas de páginas é a diferença mais visível |
| **Paginação pelo total** | Ler o `total` da primeira resposta e calcular quantas páginas faltam, em vez de ir pedindo até vir vazio |
| **Parâmetros de rede na configuração** | `timeout`, número de tentativas, espera entre tentativas e tamanho de página são parâmetros do `main_set`, não constantes no código |

Se o volume justificar paralelizar páginas, o paralelismo é dentro do notebook, no driver
(`ThreadPoolExecutor`), com o número de workers vindo também da configuração.

---

## Key Vault

Todos os segredos (SAS tokens, connection strings, API keys) são lidos do Azure Key Vault via `mssparkutils`:

```python
from dap_key_vault import read_kv_secret
secret = read_kv_secret("https://<vault>.vault.azure.net/", "nome-do-segredo")
```

Ver [Bibliotecas Comuns](bibliotecas-comuns.md#dap_key_vaultpy).

---

## Ficheiros que não se lêem por ABFSS

Bibliotecas que só sabem abrir caminhos do sistema de ficheiros — `pandas.read_excel`, `open()`,
qualquer parser que receba um path — não conseguem ler um path `abfss://`. O padrão é ler os
bytes com o Spark e entregá-los à biblioteca em memória:

```python
raw = spark.read.format("binaryFile").load(path_abfss).select("content").head()["content"]
df = pandas.read_excel(BytesIO(raw))
```

Aplica-se a XLSX, GeoJSON, ficheiros de configuração, ou qualquer binário. Evita a alternativa
que se vê com frequência — copiar para disco local, ou ler de `/lakehouse/default/` — que
depende de qual Lakehouse está afixado e parte quando isso muda.

---

## Convenção de Nomes dos Ficheiros

Os ficheiros depositados nos contentores blob devem começar com uma data no nome. A função `extract_date_from_name` suporta dois formatos:

| Padrão | Exemplo |
|--------|---------|
| `yyyyMMddHHmm_*` | `202604101516_ficheiro.csv` → `2026-04-10 15:16` |
| `yyyyMMdd_*` | `20260407_ficheiro.csv` → `2026-04-07` |

Ficheiros sem data válida no nome são ignorados com aviso `[WARN]` e incluídos na lista `skipped` devolvida por `list_files_by_date_mask_startingDate`.

A data extraída é gravada na coluna `data_ficheiro` de cada registo silver, permitindo rastrear de que extracção veio cada linha.

---

[← Bibliotecas](bibliotecas-comuns.md) · [Visão Geral](visao-geral.md) · [Home](../Home.md)
