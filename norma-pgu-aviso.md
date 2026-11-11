# Standard — PGU e Aviso de Dados

## 1. Decisões

| # | Tema | Decisão | Porquê |
|---|---|---|---|
| 1 | Pastas Bronze | Estrutura da secção 2, PGU e Aviso | Saber onde está cada ficheiro, de que período é e se passou |
| 2 | Rejeitados | Tudo o que não é `CONFORME` vai para `rejeitado/` | Nenhum ficheiro fica sem pasta nem registo |
| 3 | Entidade | Pasta = `entidade_id` da config | O mesmo id na pasta e nas tabelas |
| 4 | Tabelas | 3 tabelas comuns com coluna `dominio` | Um só report e uma só lógica |
| 5 | Língua | Português | Schema acordado a 21/08; PGU já está assim |
| 6 | Periodicidade | Os runners seguem a da config | Não ir buscar o que não mudou |
| 7 | SDTI | Fora das tabelas comuns, por agora | As tabelas de processamentos e erros existem para a validação |

## 2. Estrutura Bronze (PGU e Aviso)

```
Files/<dominio>/bronze/<entidade_id>/<conjunto_id>/validado/ano=<AAAA>/periodo=<codigo>/<ficheiro>
Files/<dominio>/bronze/<entidade_id>/<conjunto_id>/rejeitado/ano=<AAAA>/periodo=<codigo>/<ficheiro>

<ficheiro> = <periodo>_<entidade_id>_<conjunto_id>_<AAAAMMDDHHmm da ingestão>.json
```

Exemplo:

```
Files/pgu/bronze/505187531/Incident/validado/ano=2026/periodo=20260928/20260928_505187531_Incident_202609282300.json
                 └entidade┘└conj.─┘ └estado┘          └período dados┘                              └ingestão──┘
```

| Parte do exemplo | O que é |
|---|---|
| `505187531` | `entidade_id` |
| `Incident` | `conjunto_id` (na PGU é o nome do modelo) |
| `validado` / `rejeitado` | Resultado da validação |
| `ano=2026` | Ano do período |
| `periodo=20260928` | Período a que os dados se referem: diário `20260928`, mensal `202609`, anual `2026` |
| `20260928_` no nome | O mesmo período, no início do nome |
| `_202609282300` no nome | Quando nós descarregámos (28/09/2026, 23:00) |

1. Período e data de ingestão são coisas diferentes. Ex.: dados de setembro que chegam a 5 de
   outubro ficam em `periodo=202609`, com `_202610050000` no fim do nome.
2. A Bronze nunca substitui: uma correcção fica ao lado da versão anterior, na mesma pasta de período.
3. `rejeitado/` recebe `NAO_CONFORME`, `MODELO_DESCONHECIDO`, `NAO_E_DCAT`, `GRAPH_NAO_PERMITIDO`
   e `JSON_INVALIDO`.
4. Rejeitados guardam-se 30 dias.
5. Metadados: sufixo `_md` no nome (posição: secção 7).

## 3. Tabelas comuns

### 3.1 Config — uma linha por conjunto de dados de uma entidade

| Coluna | O que é |
|---|---|
| `cfg_uuid` | Chave: `uuid5` de `dominio + entidade_id + conjunto_id` |
| `dominio` | `pgu` ou `aviso_dados` |
| `entidade_id` | Id da entidade que entrega os dados (não o município a que os dados se referem) |
| `entidade` | Nome da entidade |
| `conjunto_id` | Id do conjunto de dados. Vai no caminho da Bronze. Sem espaços |
| `conjunto` | Nome do conjunto de dados |
| `tipo_fonte` | `api` ou `container` |
| `url_dados` | Endereço da API de dados |
| `header_url_dados` | Cabeçalhos HTTP que essa API exija |
| `url_metadados` | Endereço da API de metadados |
| `header_url_metadados` | Cabeçalhos HTTP que essa API exija |
| `nomenclatura_ficheiros` | **Só para `container`** (ver nota) |
| `storage_account` | Só `container`: conta Azure |
| `container` | Só `container`: nome do container |
| `periodicidade` | `daily`, `month`, `quarter`, `semester`, `year` |
| `data_inicio_ingest` | Primeira data a descarregar |
| `tipo_autenticacao` | `none`, `sas`, `api_key`, `bearer` |
| `secret_name` | Nome do segredo no Key Vault. Nunca o segredo |
| `params` | Valores a substituir no URL (ex.: data) |
| `enabled` | `false` desliga. Nunca apagar a linha |
| `created_at`, `updated_at` | Quando a linha foi criada / alterada |

**Nota — `nomenclatura_ficheiros`:**
1. API: somos nós que pedimos o ficheiro e lhe damos o nome, pela regra da secção 2. Não é
   preciso padrão.
2. Container: é a entidade que lá põe os ficheiros, com o nome que ela escolhe. O padrão (ex.
   `AAAAMMDD_Incident`) diz ao código onde está a data nesse nome.

Saem: `destino` (PGU), `landing_folder` e `ingest_end` (Aviso). O caminho segue a secção 2.

### 3.2 Processamentos — uma linha por cada vez que um ficheiro é validado

| Coluna | O que é |
|---|---|
| `processamento_uuid` | Chave `uuid4`: aleatória, nova em cada processamento. Nunca repete |
| `cfg_uuid` | Liga à config: de que entidade e conjunto é o ficheiro |
| `dominio` | `pgu` ou `aviso_dados` |
| `correlation_id` | Id da execução do runner, recebido em `log_correlation`. Nunca gerado no notebook. Os logs dessa execução têm o mesmo id: é por ele que se vai ao log ver o que aconteceu |
| `ficheiro` | Nome do ficheiro |
| `tipo_ficheiro` | `dados` ou `metadados` |
| `periodo` | **Novo.** Período dos dados (secção 2) |
| `data_ficheiro` | Primeiro dia do período, em formato data |
| `data_recebimento_ficheiro` | Quando o descarregámos |
| `estado` | Um dos 6 estados. **Nunca vazio** |
| `modelo` | Modelo do catálogo contra o qual foi validado |
| `versao_catalogo` | **Novo.** Versão do catálogo usada |
| `storage_account`, `container` | De onde veio, se veio de container |
| `path_destino` | Onde ficou no Lakehouse |
| `data_processamento` | Quando foi validado |
| `ano`, `mes`, `dia` | Do período, para filtrar. `dia` vazio se o período não for diário |

Sai `numero_erros`: o report conta as linhas da tabela de erros.

### 3.3 Erros — uma linha por problema distinto num ficheiro

| Coluna | O que é |
|---|---|
| `processamento_uuid` | Liga ao processamento |
| `propriedade` | Campo com o problema, ex. `$.dateCreated`. Se o problema é um campo em falta, o campo que falta, ex. `$.address` |
| `tipo` | `required` (falta), `format` (formato errado), `type` (tipo de dado errado), `enum` (valor fora da lista), `const` (valor diferente do único permitido), `anyOf`/`oneOf` (não bate com nenhuma forma permitida), `pattern` |
| `regra` | Caminho da regra no schema do catálogo |
| `n_registos` | Quantos registos do ficheiro têm este problema |
| `registo_exemplo` | O id de um desses registos, sem parênteses retos (ver nota) |
| `mensagem_exemplo` | A mensagem do validador para esse registo, cortada a 200 caracteres |
| `ano`, `mes`, `dia` | Do processamento |

Cada problema é uma linha: agrupar por `processamento_uuid` + `propriedade` + `tipo`. A mensagem não entra no
agrupamento. Ex.: `dateCreated` mal formatado e `address` em falta = 2 linhas.

**Nota — `registo_exemplo`:** o ficheiro que a entidade envia tem vários registos (ex. medições
de ruído). Cada registo traz um id escrito pela entidade: `id` nos dados (ex.
`urn:cmcascais:NoiseLevelObserved:CMC17:20260916T000000Z`), `@id` nos metadados. O validador não
cria ids. Guardamos um, para o report dizer à entidade qual o registo a corrigir.

## 4. Report — um xlsx por entidade, gerado por SQL sobre as tabelas da secção 3

**Folha 1 — Resumo (nova).** Uma linha por conjunto de dados da entidade, incluindo os que nunca
chegaram. Responde: entregou, a tempo, conforme, com metadados?

| Coluna |
|---|
| Conjunto |
| Período do último ficheiro de dados |
| Data em que o recebemos |
| Estado desse ficheiro |
| Dentro da periodicidade (sim/não) |
| Data do último metadado `CONFORME` |
| Nº de problemas do último ficheiro |

**Folha 2 — Problemas.** Uma linha por problema do último ficheiro de cada
conjunto. Responde: o que tem de corrigir?

| Coluna |
|---|
| Conjunto |
| Ficheiro |
| Período |
| Data em que o recebemos |
| Propriedade |
| Tipo |
| Nº de registos |
| Registo exemplo |
| Mensagem |

Entidades e conjuntos lidos da config. Corre dentro do runner.

Veredicto conforme / não conforme da entidade: secção 7.

## 5. Metadados INSPIRE

1. A lista de modelos INSPIRE é a folha do negócio ("CNMD INSPIRE"), coluna "Análise" = `INSPIRE`.
2. O bundle traz essa lista (chave `inspire`), gerada a partir da folha. Nunca editada à mão.
3. Nos metadados, o Bronze passa à lib o modelo do conjunto. Se a lib detetar `Metadata.Dataset` e
   o modelo estiver na lista, valida com `Metadata.Dataset.inspire`.
4. O Bronze não força `modelo=` na lib: isso salta a deteção do `@graph` e do `@type`.

## 6. Como se confirma que ficou bem (testes de aceitação)

1. Um ficheiro de cada um dos 6 estados cai na pasta certa e fica com linha em processamentos.
2. Processar o mesmo ficheiro duas vezes: duas linhas, duas chaves diferentes, nenhuma repetida.
3. Nenhuma linha com `estado` vazio.
4. Um ficheiro com o mesmo erro em 100 registos dá 1 linha em erros com `n_registos = 100`.
5. Um campo em falta aparece como `propriedade = $.<campo>`.
6. API desligada: fica registada a falha (depende da secção 7).
7. Um conjunto que nunca chegou aparece no resumo do report.
8. Um metadado de `Dam` sem campos `ins:` dá `NAO_CONFORME`.

## 7. Por decidir

1. Formato do `entidade_id`: hoje a PGU usa 9 dígitos (`505187531`) e o Aviso outro código (`apin15287`).
2. Posição do `_md` no nome do ficheiro.
3. Falha de API: se fica registada e com que estado.
4. Periodicidade que conta para a conformidade ("dentro da periodicidade" do report): a da config ou a do
   metadado (`dcterms:accrualPeriodicity`).
5. Veredicto conforme / não conforme da entidade.
6. INSPIRE: só os 4 campos `ins:` nos metadados, ou também os dados? Conjunto INSPIRE descrito como
   `DatasetSeries` (o catálogo não tem versão INSPIRE da série)?
