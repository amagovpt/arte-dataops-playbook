# Camada Gold — Consolidação

A camada Gold harmoniza dados de múltiplas fontes Silver, aplica mapeamentos, e produz as tabelas analíticas finais. É o destino para relatórios e modelos semânticos (Power BI).

Há duas formas de a organizar, e a escolha depende do modelo de dados, não de preferência:

| Forma | Quando | Escrita |
|---|---|---|
| **Tabela consolidada** | Várias fontes Silver com o mesmo grão, a alimentar uma tabela de factos comum | `delete` da fatia da fonte + `append` |
| **Star schema** | Modelo dimensional com factos, dimensões e pontes | Uma tabela por step, `overwrite` total em cada |

### O que é da Gold e o que não é

A Gold decide **grão, chaves e KPIs** — as regras sobre a pergunta, que mudam com quem pergunta. As
regras sobre o dado (tipos, deduplicação, geografia canónica, datas), que são verdadeiras para
qualquer consumidor, ficam na Silver.

Harmonizar fontes diferentes para o mesmo formato é trabalho de Silver ou de Gold, conforme o
alcance: fica aqui quando só esta tabela precisa dela, e sobe para uma tabela `silver_negocio`
quando outra tabela Gold vai precisar da mesma harmonização. O critério, com as duas perguntas que
o decidem, está em
[Camada Silver → Duas Zonas](camada-silver.md#como-se-organizam-as-tabelas--duas-zonas).

---

## Padrão de Escrita por Fonte: Delete + Append

O padrão principal para carregar dados de uma fonte silver para a gold:

```python
# 1. Apagar registos desta fonte
delta = DeltaTable.forPath(spark, gold_path)
delta.delete(F.col("origem").isin("fonte_a", "fonte_b"))

# 2. Fazer append dos novos registos processados
df_gold.write.format("delta").mode("append").save(gold_path)
```

**Porquê delete + append (e não overwrite)?**
- Permite que múltiplas fontes coexistam na mesma tabela gold
- Cada fonte pode ser reprocessada de forma independente
- Não afecta registos de outras fontes durante o reprocessamento

---

## Padrão de Enriquecimento: Overwrite Total

Usado para steps que relêem a tabela gold completa e adicionam colunas calculadas (ex: níveis de uma Decomposition Tree, rankings temporais):

```python
df_gold_full = spark.read.format("delta").load(gold_path)
df_gold_full = df_gold_full.join(df_mapa, chave_join, "left")

df_gold_full.write.format("delta").mode("overwrite").option("overwriteSchema", "true").save(gold_path)
```

**Quando usar overwrite total:** quando é necessário enriquecer todos os registos existentes com novas colunas ou recalcular valores com base num mapeamento actualizado.

---

## Star Schema num Único Notebook

Quando a Gold é um modelo dimensional, **todas as tabelas do modelo vivem no mesmo notebook** —
dimensões, factos e tabelas de ponte. Um step por tabela, cada um reconstruído por `overwrite`.

```
<dominio>_GOLD.ipynb
├── dim_calendario           overwrite
├── dim_<entidade>           overwrite
├── fact_<facto>             overwrite
└── ponte_<a>_<b>            overwrite
```

Porquê num só notebook e não um por tabela:

1. As tabelas partilham leituras da Silver e funções de harmonização. Separá-las obriga a repetir
   as duas coisas ou a inventar uma camada intermédia.
2. A ordem entre elas importa (as dimensões antes dos factos que lhes chamam), e num notebook
   único a ordem é a das células, não configuração.
3. O `running_steps` já dá o controlo fino de correr uma tabela isolada.

`overwrite` é aceitável aqui porque cada tabela é **reconstruída de raiz** a partir da Silver a
cada execução — não há histórico a preservar. Uma tabela de factos que acumule histórico próprio
não segue este padrão; segue o `delete` + `append` acima. Ver
[Princípios → Criar a tabela e escrever nela são operações diferentes](principios.md#criar-a-tabela-e-escrever-nela-são-operações-diferentes).

**O `overwriteSchema` acrescenta-se onde o schema muda entre execuções, não por defeito.** Sem
ele, uma escrita cujo schema divergiu do da tabela falha, e obriga a olhar para a alteração. Com
ele, a tabela é substituída em silêncio pelo schema novo — uma coluna que desapareceu por engano
da Silver desaparece da Gold sem aviso nenhum. Pô-lo em todas as escritas troca uma falha visível
por uma perda silenciosa.

---

## Mapeamentos Externos

Valores categóricos vindos de fontes diferentes normalizam-se por tabela de correspondência —
um ficheiro no Lakehouse quando o mapa é mantido pelo negócio, um dicionário no notebook quando é
fixo e pequeno.

| Regra | Porquê |
|--------|-----------|
| Join sempre `left` | Um valor sem correspondência fica a `null` no campo mapeado, mas **o registo nunca é descartado**. Perder linhas num join é o erro mais caro desta camada, porque é silencioso |
| Valor por defeito explícito | `F.coalesce(F.col("mapeado"), F.lit("<default>"))` — o `null` do `left join` tem de ser tratado, senão aparece no relatório |
| Mapa fora do código quando muda | Se quem mantém o mapa é o negócio, o mapa é um ficheiro. Se é uma constante técnica, é um dicionário no notebook |

---

## Steps Configuráveis

Um notebook Gold constrói várias tabelas. `running_steps` é a lista das que se querem construir
nesta execução: cada bloco começa por verificar se está na lista e, se não estiver, não faz nada.

```python
try:
    if "dim_organizacao" in steps:
        ...
        df.write.format("delta").mode("overwrite").option("overwriteSchema", "true").save(path)
        results.append({"step": "dim_organizacao", "rows": df.count()})
except Exception as e:
    handleError(120, e, running_local, results)
```

Serve para reprocessar uma tabela isolada sem correr o notebook todo — em desenvolvimento, ou
depois de corrigir uma fonte. A lista vem do `main_set`; onde exactamente depende da geração do
runner ([ver Orquestração](orquestracao-runner.md#notas-sobre-notebooks-gold-com-steps-configuráveis)).

**`running_steps` não é o `movement_type`.** São coisas independentes: o `movement_type` descreve
a intenção da task no ficheiro de configuração e não é lido pelo runner actual; o `running_steps`
é lido pelo notebook e decide o que corre. Um notebook Gold pode ser invocado por duas tasks
diferentes, com `running_steps` diferentes, para fazer trabalho diferente.

---

## Schema Típico da Tabela Gold

A gold expõe um schema harmonizado e normalizado, independente das diferenças entre as fontes silver:

- Datas em `DateType` ou `TimestampType`
- Valores numéricos em `DoubleType` ou `LongType`
- Strings normalizadas (sem acentos, uppercase quando relevante para joins)
- Coluna `origem` identifica a fonte de cada registo
- Colunas de auditoria herdadas da silver

---

## Resultado Devolvido ao Runner

```json
{
  "status": "OK",
  "results": [
    { "step": "dim_<entidade>", "rows": 1234 },
    { "step": "fact_<facto>",   "rows": 56789 }
  ]
}
```

O `results` é acumulado célula a célula e passado ao `handleError` em cada `except`. Se um step
falhar a meio, o runner recebe na mesma o registo dos que já tinham corrido.

---

[← Camada Silver](camada-silver.md) · [Orquestração →](orquestracao-runner.md) · [Home](../Home.md)
