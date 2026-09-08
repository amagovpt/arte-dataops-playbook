# Runbook — Um Pipeline Falhou

Sequência para diagnosticar e recuperar uma execução falhada. Todas as peças estão documentadas
nas outras páginas; esta põe-nas por ordem para quem tem um pipeline em baixo.

---

## 1. Encontrar a execução

Cada execução do runner gera um `correlation_id` no formato `yyyyMMddHHmmss`, que é propagado a
todos os notebooks-filho. É a chave para ver a execução inteira, e não um notebook isolado.

```sql
SELECT correlation_id, MIN(timestamp) AS inicio, MAX(timestamp) AS fim
FROM common_lh_observability.ELogger.pipeline_logs
WHERE pipeline_id = '<pipeline>'
GROUP BY correlation_id
ORDER BY inicio DESC
LIMIT 10
```

---

## 2. Ver o que correu e onde parou

```sql
SELECT timestamp, notebook, step_name, level, message
FROM common_lh_observability.ELogger.pipeline_logs
WHERE correlation_id = '<correlation_id>'
ORDER BY timestamp
```

O que procurar, por ordem:

| Sinal | Significa |
|---|---|
| Linhas com `level = 'error'` | O erro propriamente dito, com o `errorState` na mensagem |
| Último `step_name` antes do erro | Em que passo do notebook parou |
| Ausência de um `Finish` | O notebook não chegou ao fim |
| Um notebook que nunca aparece | Foi saltado, não falhou. Ver o ponto 4 |

O `errorId` no JSON devolvido diz o passo dentro do notebook. Os números são locais a cada
notebook — ler a tabela da página da camada correspondente.

---

## 3. Quando não há logs nenhuns

Se a tabela não tem sequer o `Start` do runner, o notebook falhou antes de conseguir logar. Duas
causas prováveis, por esta ordem:

1. **O import das Global Libs falhou.** Numa execução pelo runner, o `/lakehouse/default/` vem da
   sessão do **runner**, não do notebook-filho. O `%%configure` a corrigir é o do runner
   ([ver Orquestração](orquestracao-runner.md#a-célula-configure)).
2. **A configuração não foi lida.** O `read_config_pipelines` procura o ficheiro pelo nome exacto
   em `Files/configs/pipelines/`. Um nome trocado dá erro logo na primeira célula.

---

## 4. Um notebook que não correu, sem erro nenhum

Não é falha, é `SKIP`. Três causas, e nenhuma produz erro:

| Causa | Verificar |
|---|---|
| `enabled` a `false` no grupo ou na task | O JSON de configuração |
| `enabled` como **string** `"True"` em vez de booleano | Uma string nunca iguala `1` na comparação do runner, e a task é ignorada em silêncio |
| `new_data = 0` e `run_withouth_new_data = false` | Que valor o notebook anterior devolveu. **Todas** as tasks reescrevem o `new_data`, não só as de ingestão |

---

## 5. Re-executar só o que falhou

Nunca é preciso correr o pipeline todo outra vez.

| Alcance | Como |
|---|---|
| Uma camada | No runner, `local_updates` para desligar os grupos que já correram, sem tocar no ficheiro de configuração |
| Uma tabela Gold | Abrir o notebook Gold, `running_local = True`, e deixar em `running_steps` só o step a reconstruir |
| Um notebook isolado | Abrir, pôr `running_local = True`, ajustar a célula `parameters` e correr |

Com `running_local = True` um erro levanta excepção com o traceback completo, em vez de terminar o
notebook em silêncio.

---

## 6. Antes de re-executar, confirmar que é seguro

| Padrão da camada | É seguro repetir? |
|---|---|
| Bronze consume-and-delete | **Verificar primeiro.** Se o blob já foi apagado da origem, não há o que voltar a copiar |
| Bronze por API | Sim — é um full refresh |
| Silver `overwrite` | Sim |
| Gold `overwrite` | Sim |
| Gold `delete` + `append` | Sim, desde que o `delete` filtre a fatia certa |
| Gold `append` puro | **Não sem apagar antes.** Repetir duplica linhas |

---

## 7. Se a execução foi muito mais lenta do que o costume

Verificar primeiro se há `elogger.log_event_*` dentro de um ciclo. Cada chamada é um job Spark e um
commit Delta, com custo fixo independente do volume, e domina o tempo do notebook a partir de
algumas dezenas de iterações ([ver ELogger](bibliotecas-comuns.md#custo--nunca-logar-dentro-de-um-ciclo)).

---

[Orquestração](orquestracao-runner.md) · [Bibliotecas Comuns](bibliotecas-comuns.md) · [Home](../Home.md)
