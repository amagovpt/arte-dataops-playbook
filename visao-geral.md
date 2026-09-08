# Visão Geral — Metodologia de Pipelines Fabric

Esta metodologia define os padrões para construir e organizar pipelines de dados em Microsoft Fabric.

---

## Arquitectura Medallion

Todos os pipelines seguem a arquitectura **Bronze → Silver → Gold**:

```
Fonte Externa
     │
     ▼
  BRONZE          Cópia fiel da fonte — sem transformações de schema
     │
     ▼
  SILVER          Normalização: cast de tipos, colunas de auditoria, overwrite
     │
     ▼
  GOLD            Consolidação: harmonização de múltiplas fontes, mapeamentos, enriquecimento
```

### Quando usar cada camada

| Camada | Responsabilidade | Escrita |
|--------|-----------------|---------|
| **Bronze** | Copiar dados externos para o Lakehouse sem alterar o conteúdo | Ficheiros originais em `Files/`, ou tabela Delta em `Tables/` com o payload tal como veio |
| **Silver** | Normalizar schema, cast de tipos, colunas de auditoria e, quando mais do que uma tabela Gold precisa dela, harmonizar fontes para o mesmo formato | Delta, mode=overwrite |
| **Gold** | Decidir grão, chaves e métricas: consolidar, mapear, enriquecer | Delta, delete+append por fonte ou overwrite total |

---

## Organização dos Notebooks

Cada pipeline tem uma pasta de notebooks com esta estrutura típica:

```
notebooks/<pipeline>/
├── <pipeline>_runner.ipynb        — orquestrador, único ponto de entrada
├── <pipeline>_BRONZE.ipynb        — ingestão (ou vários notebooks de bronze por fonte)
├── <pipeline>_SILVER.ipynb        — normalização (ou vários por fonte)
├── <pipeline>_GOLD.ipynb          — consolidação
└── <pipeline>.json                — configuração lida pelo runner em runtime
```

O `.json` não é importado como notebook: é copiado para `Files/configs/pipelines/` do Lakehouse de configuração, de onde o runner o lê ([ver Orquestração](orquestracao-runner.md#onde-vive-o-ficheiro)).

As bibliotecas partilhadas são compiladas a partir dos notebooks em `notebooks/Utils/` e ficam em `Files/Global_Libs/` do Lakehouse de **suporte** — que é outro Lakehouse, e é aquele que os notebooks declaram no `%%configure`.

---

## Padrões Transversais

Todos os notebooks do pipeline partilham os mesmos padrões:

- **Parâmetros via `main_set`** — configuração injectada pelo runner em runtime ([ver Orquestração](orquestracao-runner.md))
- **Logging estruturado** — ELogger com `correlation_id` propagado entre notebooks ([ver Bibliotecas](bibliotecas-comuns.md))
- **Error handling** — padrão `handleError` com saída imediata em produção ([ver Bibliotecas](bibliotecas-comuns.md#handleerror))
- **Execução local vs. produção** — flag `running_local` controla comportamento de erros e parâmetros

---

## Decisões Arquitecturais

### Sem Vendor Lock-in — Notebooks em vez de Data Factory / Dataflow Gen2

Todos os pipelines são implementados em **Spark notebooks**, não em Data Factory Gen2 pipelines nem Dataflow Gen2. Razões:

- Os notebooks são portáteis — correm localmente, num workspace de desenvolvimento, e em produção sem alterações
- A lógica está em Python/PySpark, não em configuração visual proprietária que não é versionável de forma útil
- O runner invoca notebooks via `notebookutils.notebook.run()` — sem dependências de conectores proprietários
- Sem componentes visuais cujo comportamento muda entre versões do Fabric sem aviso

### Tudo Parametrizável — Sem Valores Hardcoded

**Nunca** se hardcodam paths de Lakehouses, nomes de contentores, credenciais, ou qualquer configuração que varie por ambiente. Tudo passa via `main_set`:

```python
# ✅ Correcto — parametrizado via main_set
lh_bronze = main_set["lh_bronze"]   # abfss://<workspace_guid>@onelake.dfs.fabric.microsoft.com/<lakehouse_guid>

# ❌ Errado — hardcoded
lh_bronze = "abfss://33f78fb2-abc-.../<lakehouse_guid>"
```

Isto garante que o mesmo notebook corre em desenvolvimento, staging e produção apenas mudando os parâmetros no runner.

### Sempre ABFSS — Sem Montagem de Lakehouses

Os paths de Lakehouse são sempre em formato **ABFSS** (`abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>/`). Nunca se "afixa" um Lakehouse em construção de paths — os paths chegam sempre como parâmetro.

**GUID ou nome, nunca os dois.** Com GUIDs o path não leva sufixo:

```
abfss://<workspace_guid>@onelake.dfs.fabric.microsoft.com/<lakehouse_guid>/Files/...
```

O sufixo `.Lakehouse` só acompanha o **nome** (`meu_lh.Lakehouse`). Juntar GUID e sufixo dá `Operation failed: "Bad Request", 400` no primeiro acesso — e não dá erro nenhum na escrita do notebook, só em runtime. Todos os notebooks do repo usam a forma com GUID sem sufixo.

A excepção são as **Global Libs**, que vivem no Lakehouse de **suporte**, declarado como `defaultLakehouse` na célula `%%configure -f` do notebook. Isto permite o import directo:

```python
import sys
sys.path.append("/lakehouse/default/Files/Global_Libs")
from dap_ELogger import ELogger
```

Este é o **único** `/lakehouse/default/` legítimo, e funciona por causa do `%%configure` — não por alguém ter afixado o Lakehouse pela UI do Fabric. O Lakehouse de suporte nunca é o Lakehouse de dados do pipeline, e as libs nunca se copiam para lá. Para tudo o resto, usar ABFSS explícito derivado do `main_set`.

### Separação de Workspaces por Tipo de Acesso

| Workspace | Quando usar |
|-----------|------------|
| `e_ext_ingest_ws` | Dados com acesso **público** ou provenientes do **exterior** (APIs públicas, parceiros, upload de ficheiros) |
| `e_int_ingest_ws` | Dados com acesso **privado** ou de sistemas internos |

Pipelines que consomem APIs públicas ou ficheiros de parceiros externos correm em `e_ext_ingest_ws`. Pipelines que acedam a sistemas internos ou dados sensíveis devem usar `e_int_ingest_ws`.

---

## Trabalho Futuro

Áreas identificadas para evolução, mas ainda não implementadas:

- **Funções centralizadas:** as funções de cast e normalização estão definidas dentro de cada notebook Silver e diferem de pipeline para pipeline. Enquanto for assim, ao escrever um notebook novo é preciso ler as funções do notebook de onde se parte — não se pode assumir que um tipo do `schema_map` se comporta da mesma maneira em dois pipelines ([ver Camada Silver](camada-silver.md#as-funções-de-cast)).
- **Configuração em tabelas Delta:** a configuração dos pipelines já saiu dos notebooks para ficheiros JSON no Lakehouse ([ver Orquestração](orquestracao-runner.md)). O passo seguinte é passá-la a tabelas Delta, para se poder alterar por SQL e manter histórico. Ainda não está feito em nenhum pipeline.

---

## Documentação da Metodologia

| Secção | Conteúdo |
|--------|----------|
| [Princípios Arquitecturais](principios.md) | As regras que decidem tudo o resto — escrita por path, formato aberto, portabilidade |
| [Orquestração — Runner](orquestracao-runner.md) | Configuração declarativa, anatomia dos notebooks, injecção de parâmetros |
| [Camada Bronze](camada-bronze.md) | Padrões de ingestão de dados externos |
| [Camada Silver](camada-silver.md) | Normalização, casting, auditoria |
| [Camada Gold](camada-gold.md) | Consolidação, mapeamentos, enriquecimento |
| [Bibliotecas Comuns](bibliotecas-comuns.md) | ELogger, `handleError`, Azure REST, Lakehouse file tools |
| [Fontes de Dados](fontes-de-dados.md) | Padrões de acesso a fontes externas |
| [Runbook](runbook.md) | O que fazer quando um pipeline falha |
| [Checklists](checklists/checklist-bronze.md) | Verificação de conformidade antes de passar a produção — [Bronze](checklists/checklist-bronze.md), [Silver](checklists/checklist-silver.md), [Gold](checklists/checklist-gold.md), [Runner](checklists/checklist-runner.md) |

Para ver tudo isto aplicado a um pipeline real, de ponta a ponta:
[dados.gov.pt](exemplo-dados-gov.md).

---

[Voltar à Home](../Home.md)
