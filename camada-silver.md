# Camada Silver — Normalização

A camada Silver lê dados do Bronze, normaliza o schema, faz cast de tipos, adiciona colunas de auditoria e escreve uma tabela Delta. A escrita é sempre **overwrite** — a tabela silver é substituída na íntegra a cada execução.

---

## Como se Organizam as Tabelas — Duas Zonas

A pergunta aparece assim que um pipeline tem mais do que uma fonte: as tabelas Silver são uma por
sistema de origem, ou uma por conceito de negócio já harmonizado?

São as duas, em zonas separadas do mesmo Lakehouse:

| Zona | O que é |
|---|---|
| `silver_origem.<fonte>` | Uma tabela por fonte, 1:1 com o sistema de origem: `schema_map`, cast, limpeza, auditoria |
| `silver_negocio.<entidade>` | Fontes já limpas, postas no mesmo formato e tratadas como uma só |

A `silver_origem` é o que já se faz e não muda. A `silver_negocio` é **tabela extra e opcional**:
acrescenta-se, não substitui a primeira. E tem limites estritos — **mesma linha, mesmo grão, sem
chaves substitutas, sem agregação, sem KPIs.** Harmonizar não é modelar.

### Quando criar uma `silver_negocio`

Duas perguntas, por esta ordem:

1. **A tabela Gold junta linhas de vários sistemas de origem?**
2. **Se sim, outra tabela Gold vai precisar dessa mesma junção?**

Não à 2 → fica no Gold. É o caso normal.  
Sim à 2 → sai para `silver_negocio.<entidade>`.

Com um único Gold não há ganho nenhum: os mapeamentos por fonte continuam a existir na mesma, só
mudam de sítio.

### O que se evita

A mesma regra de negócio escrita em dois lados.

```
Antes                                              Depois
GOLD A: fonte_1 + fonte_2 → harmonizar → fact_X    SILVER: fonte_1 + fonte_2 → harmonizar → silver_negocio.entidade
GOLD B: fonte_1 + fonte_2 → harmonizar → fact_Y    GOLD A: silver_negocio.entidade → fact_X
                                                   GOLD B: silver_negocio.entidade → fact_Y
```

Muda um código numa das fontes: à esquerda corrigem-se dois sítios, e eles podem divergir sem que
ninguém dê por isso; à direita corrige-se um e os dois ficam certos.

**A fronteira, em uma linha:** pôr fontes no mesmo formato é regra sobre o dado, e é Silver; grão,
chaves e KPIs são regra sobre a pergunta, e são [Gold](camada-gold.md).

---

## Fluxo Padrão

```
1. Receber parâmetros via main_set
        ↓
2. Seleccionar ficheiro a processar (ver secção abaixo)
        ↓
3. Ler dados (CSV, Delta, API, etc.)
        ↓
4. Normalizar schema: cast de cada coluna segundo schema_map
        ↓
5. Limpar NULLs: substituir strings "NULL" (case-insensitive) por null real
        ↓
6. Adicionar colunas de auditoria
        ↓
7. df.write.format("delta").mode("overwrite").save(silver_path)
        ↓
8. Retornar { "status": "OK", "records_written": N, "new_data": 1 }
```

**Atenção ao `new_data` na Silver.** O runner reescreve o `new_data` com o resultado de *cada*
task, não só das de ingestão, e assume `1` se a chave faltar. Uma Silver que devolva `new_data: 0`
faz a Gold ser saltada, mesmo que o Bronze tenha trazido dados novos. Devolver `1`, ou não
devolver a chave, a menos que se queira mesmo travar o que vem a seguir
([ver Orquestração](orquestracao-runner.md#new_data)).

---

## Selecção do Ficheiro a Processar

Quando a bronze são ficheiros com data no nome, a selecção do mais recente é automática:

### Modo 1 — Ficheiro explícito
Se `nome_ficheiro != ""`, o notebook usa esse nome directamente.

### Modo 2 — Selecção automática (modo normal de produção)
Se `nome_ficheiro == ""` (valor por defeito), o notebook chama `list_files_by_date_mask_startingDate`:

```python
matched, skipped = list_files_by_date_mask_startingDate(caminho_bronze, mask_ficheiro)
if len(matched) > 0:
    nome_ficheiro = matched[0]['name']    # ficheiro mais recente
    data_ficheiro = matched[0]['file_date']
```

A data é extraída do início do nome do ficheiro — ver [Fontes de Dados](fontes-de-dados.md#convenção-de-nomes-dos-ficheiros).

---

## Colunas de Auditoria

**Estado real: não há convenção cumprida.** Cada pipeline em produção adiciona um conjunto
diferente, e um deles quase não adiciona nada. Quem lê uma tabela Silver não pode assumir que
encontra estas colunas.

O mínimo a exigir num pipeline novo:

| Coluna | Tipo | Conteúdo |
|--------|------|----------|
| `origem` | `string` | Identificador da fonte, o mesmo valor que a Gold usa para apagar e reescrever a fatia desta fonte |
| `dt_ingest` | `timestamp` | `current_timestamp()` no momento da execução Silver |

E, só quando a Bronze são ficheiros com data no nome:

| Coluna | Tipo | Conteúdo |
|--------|------|----------|
| `ficheiro_origem` | `string` | Nome do ficheiro lido |
| `data_ficheiro` | `date` | Data extraída do nome do ficheiro |

O `origem` é o que torna o padrão `delete` + `append` da Gold possível ([ver Camada Gold](camada-gold.md#padrão-de-escrita-por-fonte-delete--append)). Sem ele, uma tabela Gold alimentada por várias fontes não se consegue reprocessar por partes.

---

## Normalização por `schema_map`

A normalização é metadata-driven: uma lista declara a correspondência entre a origem e a
tabela final, e uma função genérica aplica-a. A lógica não conhece nomes de colunas.

```python
schema_map = [
    {"input": "id",              "output": "dataset_id", "type": "string"},
    {"input": "organization.id", "output": "org_id",     "type": "string"},
    {"input": "created_at",      "output": "created",    "type": "date"},
]

df = normalizar_colunas(df_bronze, schema_map)
```

| Chave | Para quê |
|---|---|
| `input` | Nome do campo na Bronze. Suporta notação com ponto quando a Bronze preserva JSON aninhado |
| `output` | Nome da coluna na Silver |
| `type` | Tipo alvo, que escolhe a função de cast |

Um `input` que não exista gera a coluna a `null` com aviso, em vez de rebentar. É deliberado:
uma fonte que deixa de enviar um campo não deve parar o pipeline.

### As funções de cast

**Estão definidas dentro de cada notebook Silver, não numa biblioteca.** Consequência directa:
o conjunto de tipos suportados e o comportamento de cada um **variam de pipeline para pipeline**,
e não há garantia nenhuma de que `type: "double"` signifique o mesmo em dois notebooks
diferentes. Ao escrever um notebook novo, ler as funções do notebook de que se está a partir —
não assumir.

Centralizar estas funções está identificado como trabalho futuro ([ver Visão Geral](visao-geral.md#trabalho-futuro)).
Enquanto isso não acontece, o que segue é o **contrato a implementar** num notebook novo. Não
descreve o que todos os notebooks existentes fazem — descreve o que passam a ter de fazer, para que
o mesmo `type` signifique o mesmo em dois sítios, e para que no dia em que forem centralizadas já
haja uma assinatura definida.

### Contrato dos tipos

Duas regras valem para todos, sem excepção:

1. **Um cast que não consegue converter devolve `null`.** Nunca levanta excepção. Uma linha com um
   valor ilegível passa com o campo vazio; não trava as outras dez mil.
2. **Um `type` desconhecido no `schema_map` avisa.** Não passa em silêncio deixando a coluna com o
   tipo que vinha da origem.

| `type` | Devolve | Contrato |
|---|---|---|
| `string` | `StringType` | Cast directo |
| `int` | `IntegerType` | Descarta separadores de milhares e caracteres não numéricos antes do cast |
| `long` | `LongType` | Igual ao `int`, para valores que não cabem em 32 bits |
| `double` | `DoubleType` | Reconhece formato português (`1.234,56`) e inglês (`1,234.56`). Quando o valor é ambíguo, a convenção do pipeline tem de estar declarada, não adivinhada linha a linha |
| `boolean` | `BooleanType` | Cast directo. A fonte já traz booleanos verdadeiros ou não traz nenhuns |
| `date` | `DateType` | Aceita uma lista de formatos declarada no `schema_map`, tentados por ordem |
| `timestamp` | `TimestampType` | Igual ao `date`, preservando a hora |
| `raw` | Inalterado | Arrays e structs que a Gold vai precisar inteiros. Não fazer cast é a escolha, não a omissão |

Casos que a fonte imponha e que este contrato não cubra resolvem-se com um tipo próprio, nomeado
pelo que faz e documentado no notebook. Não se alarga o significado de um tipo existente. O caso
mais comum é o Excel, que grava datas como número de série — precisa de um tipo à parte, porque um
`cast` normal lê o número como número.

---

## Tratamento de Erros

Existem dois padrões em uso — o padrão novo é o de referência.

### Padrão Novo — `handleError` (referência)

`handleError` vem das Global Libs. **Não devolve nada**: interrompe sempre a execução, saindo do
notebook em produção e levantando excepção em local. Por isso cada passo é um `try`/`except`
independente, sem guardas de `errorId` à volta:

```python
try:
    df = normalizar_colunas(df, schema_map)
except Exception as e:
    handleError(40, e, running_local, {"status": "normalizar_colunas"})
```

O comportamento exacto, e uma ressalva da plataforma sobre chamar `notebook.exit()` dentro de um
`except`, estão em [Bibliotecas Comuns](bibliotecas-comuns.md#handleerror). Ler antes de copiar
o padrão.

### Padrão Antigo — verificação inline

Define o `handleError` dentro do próprio notebook, numa versão que **devolve** `(errorState, errorId)`
em vez de interromper, e guarda cada passo com `if errorId == 0:`. O notebook continua a correr
depois do erro e só termina no fim. Mais difícil de depurar, e a origem da confusão entre as duas
assinaturas. Notebooks com este padrão devem migrar para o `handleError` da biblioteca.

---

## Códigos de Erro

Exemplo de atribuição numa Silver, seguindo a ordem das células:

| `errorId` | Situação |
|-----------|----------|
| `0` | Sucesso |
| `10` | Falha na configuração inicial / parâmetros |
| `20` | Falha a resolver os paths ou a encontrar o ficheiro na Bronze |
| `30` | Falha na leitura da Bronze |
| `40` | Falha na normalização de colunas |
| `50` | Falha nas colunas derivadas e de auditoria |
| `60` | Falha na escrita Delta |

Não é um contrato entre pipelines — ver [Orquestração → `errorId`](orquestracao-runner.md#errorid).

---

[← Camada Bronze](camada-bronze.md) · [Camada Gold →](camada-gold.md) · [Home](../Home.md)
