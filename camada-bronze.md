# Camada Bronze — Ingestão

A camada Bronze copia dados de fontes externas para o Lakehouse **sem transformar** o schema ou o conteúdo. É a landing zone fiel da fonte.

---

## Responsabilidade

- Copiar ficheiros ou dados de fontes externas para o Lakehouse Bronze
- Preservar o ficheiro/dado original sem alterações de schema
- Registar metadados da ingestão (nome do ficheiro, timestamp)
- Sinalizar ao runner se existiram dados novos (`new_data`)

**Onde aterra, conforme a fonte:**

| Fonte | Destino | Porquê |
|---|---|---|
| Ficheiros (CSV, XLSX, JSON em blob) | `Files/<folder_destino>/` | O ficheiro é copiado tal como está. Converter já seria transformar |
| Resposta de API (JSON) | `Tables/<schema>/<tabela>` em Delta | Não há ficheiro original a preservar. A estrutura aninhada é preservada como está, sem achatar |

Quando o destino é uma tabela, acrescentam-se colunas técnicas de proveniência — tipicamente o
instante da ingestão e o endpoint de origem. Quando o destino é um ficheiro, não se acrescenta
nada: a proveniência é o próprio nome do ficheiro e a pasta onde aterrou.

Em qualquer dos casos, **nada mais é acrescentado ou alterado**. Uma coluna de proveniência não
interpreta nenhum campo da fonte. Qualquer coisa que corrija, complete ou reinterprete um valor é
Silver, mesmo que seja tentador fazê-la aqui.

---

## Padrões de Ingestão

**Regra geral: tudo corre em notebook PySpark.** A única excepção é quando a fonte só é alcançável através do gateway (PBIGW) — o Fabric corre na cloud Azure e um notebook não consegue atravessar esse gateway directamente:

- **Base de dados on-premises** → só é possível via **Fabric CopyJob** (Tipo 5)
- **API interna acedida via VPN** → precisa de **Web Activity** do Fabric Pipeline para fazer a chamada (Tipos 3/4) — só as activities do pipeline conseguem usar essa ligação de gateway

Fora destes dois casos (fonte pública ou acessível directamente pela rede do Fabric), a ingestão corre inteiramente no notebook, incluindo quando há autenticação (Key Vault, SAS, etc.) — ver Tipos 1 e 2.

Existem 5 tipos, conforme a fonte e a necessidade (ou não) de gateway. Ver também [Checklist Bronze](checklists/checklist-bronze.md) para os requisitos de conformidade de cada um.

Nos tipos que precisam de gateway o pipeline passa a ter **dois artefactos**, não só o notebook: nos Tipos 3 e 4 é a Fabric Pipeline com as `WebActivity`; no Tipo 5 é o CopyJob, um item do workspace que o notebook dispara por REST.

Quando a definição da Fabric Pipeline está versionada no repositório, é um JSON em formato ARM (`$schema`, `contentVersion`, `resources`). **Não confundir com o ficheiro de configuração do runner**, que tem `workspace`, `pipeline_id`, `allTasks` e `groups`. Dois JSON, papéis diferentes.

### Tipo 1 — Azure Blob Storage (consume-and-delete)

Usado quando a fonte deposita ficheiros num contentor blob e o pipeline é o único consumidor. Corre inteiramente dentro do notebook:

```
1. Autenticar → obter SAS Token do Key Vault
2. Listar blobs no contentor (list_blobs_via_rest)
3. Para cada blob:
   a. Descarregar (streaming, sem carregar em memória)
   b. Fazer upload para o Lakehouse em chunks de 4 MB (ver abaixo)
   c. Apagar o blob na origem (só após upload confirmado)
4. Retornar { files_copied: [...] }
```

**Propriedade chave:** se o upload falhar, o blob NÃO é apagado e fica disponível para reprocessamento na próxima execução.

**Upload por chunks (Azure Data Lake Storage Gen2 REST API):** para ficheiros grandes, o upload ao destino é feito em chunks de **4 MB**, chamado directamente do notebook via `requests` (sem SDK):

1. `PUT` — cria o ficheiro vazio no destino (`?resource=file`)
2. `PATCH` em loop — anexa chunks (`?action=append&position=N`)
3. `PATCH` final — confirma a escrita (`?action=flush&position=total`)

Autenticação: Bearer Token via `mssparkutils.credentials.getToken("https://storage.azure.com/.default")`. Implementado em `dap_lake_file_tools.py` ([ver Bibliotecas](bibliotecas-comuns.md#dap_lake_file_toolspy)).

### Tipo 2 — REST API pública (directa)

Usado quando a fonte expõe uma API REST pública, sem autenticação. Corre inteiramente dentro do notebook, via `requests` directo:

```
1. Descobrir o catálogo de dados disponíveis (endpoint de metadata)
2. Para cada item do catálogo:
   a. Fazer GET ao endpoint de dados
   b. Parsear a resposta (CSV, JSON, etc.)
   c. Escrever em tabela Delta no Lakehouse Bronze
3. Retornar { records_loaded: N, new_data: bool }
```

### Tipo 3 — REST API interna (via VPN) com OAuth2, via Fabric Pipeline

Usado quando a fonte é uma API interna só alcançável via gateway (PBIGW), com autenticação OAuth2 (client credentials). Por isso a chamada à API **não corre no notebook** — corre em activities `WebActivity` do Fabric Pipeline, que fazem o pedido de token e a chamada à API antes de invocar o notebook:

```
1. Fabric Pipeline, activity WebActivity "pedido_token":
   POST ao endpoint de token com client_id/client_secret (secureInput: true)
2. Fabric Pipeline, activity WebActivity "chamada_api":
   GET ao endpoint de dados com "Authorization: Bearer <token>"
3. Fabric Pipeline invoca o notebook (TridentNotebook), passando o resultado da API já como parâmetro
4. O notebook só grava o resultado recebido no Lakehouse (mssparkutils.fs.put) — não faz nenhuma chamada de rede
```

O `client_secret` é lido do Key Vault e passado como parâmetro seguro ao pipeline. Lógica incremental (histórico → YTD → incremental) é implementada nos parâmetros de data do pipeline.

**Quem dispara a pipeline é um notebook.** O runner invoca um notebook Bronze normal, que lê o segredo do Key Vault, dispara a Fabric Pipeline por REST (`sempy.fabric.FabricRestClient`) e faz polling até ela terminar. São por isso **dois** notebooks: o que orquestra, chamado pelo runner, e o pequeno que a pipeline invoca no fim só para gravar o resultado. O contrato do runner mantém-se — quem lhe responde é o notebook que orquestra.

### Tipo 4 — REST API interna (via VPN) com Basic Auth, via Fabric Pipeline

Mesmo esquema do Tipo 3 (API interna, só alcançável via gateway), mas com Basic Auth: o notebook lê a password do Key Vault e constrói o `auth_header` (`"Basic " + base64(user:password)`), que é passado como parâmetro seguro à `WebActivity` do Fabric Pipeline — é essa activity que faz a chamada GET/POST à API. O notebook seguinte só recebe e grava o resultado.

### Tipo 5 — Base de Dados On-Premises, via Fabric CopyJob

Usado para fontes de base de dados on-premises — a única forma suportada de lá chegar a partir do Fabric é o CopyJob (autenticação por managed identity do Fabric, sem Key Vault). O notebook dispara o CopyJob e espera pela conclusão, devolvendo o status final (`Succeeded`/`Completed`).

Quando a origem é uma base de dados já modelada, o CopyJob aterra dados que dispensam normalização e o pipeline salta a Silver, indo do Bronze directamente para a Gold (`movement_type: bronze_to_gold`). Não é uma propriedade do Tipo 5, é uma consequência da fonte: se os dados precisarem de normalização, a Silver faz falta na mesma.

---

## Resultado Devolvido ao Runner

O notebook Bronze retorna sempre um JSON com o estado da execução, e esse JSON **tem de conter
`new_data`**:

```json
{ "status": "OK", "errorId": 0, "new_data": 1,
  "files_copied": [ { "name": "ficheiro.csv", "created_on": "..." } ] }
```

**Quem decide se há dados novos é o notebook Bronze.** O runner limita-se a ler a chave
`new_data` e a agir sobre ela: se vier a `0`, os grupos seguintes só correm se tiverem
`run_withouth_new_data = true`.

O runner não olha para mais nenhum campo. Não conta os `files_copied`, não compara contagens, não
inspecciona o resultado. Se o notebook devolver `new_data: 1` sem ter copiado nada, a Silver e a
Gold correm; se devolver `0` depois de copiar mil ficheiros, não correm. **A responsabilidade de
pôr o valor certo é inteiramente do notebook.**

> Nos runners da geração antiga era o runner que derivava o `new_data` da contagem de ficheiros
> copiados. Deixou de ser assim ([ver Orquestração](orquestracao-runner.md#as-duas-gerações-de-runner)).

---

## Tratamento de Erros

Cada passo em `try`/`except` chama `handleError` com um `errorId` crescente. Exemplo de uma
atribuição típica num Bronze:

| `errorId` | Situação |
|-----------|----------|
| `0` | Sucesso |
| `10` | Falha na leitura da configuração / parâmetros |
| `20` | Falha de autenticação ou acesso à fonte |
| `30`+ | Falha na cópia ou escrita, um código por passo |

Os números são locais ao notebook e não constituem contrato entre pipelines — ver
[Orquestração → `errorId`](orquestracao-runner.md#errorid). O comportamento do `handleError`
está em [Bibliotecas Comuns](bibliotecas-comuns.md#handleerror).

---

[Visão Geral](visao-geral.md) · [Camada Silver →](camada-silver.md) · [Home](../Home.md)
