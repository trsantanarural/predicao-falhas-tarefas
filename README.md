# Predição de Falhas em Tarefas em Ambientes de Computação em Nuvem

Código do artigo desenvolvido na disciplina Análise de Dados e Aprendizado de Máquina (PPGIA/UFRPE).

O trabalho prevê, com **10 minutos de antecedência**, se a primeira tentativa de execução de uma tarefa no Google Cluster Data 2011 terminará em falha (FAIL) ou em conclusão normal (FINISH). São usadas duas etapas de aprendizado de máquina sobre a mesma base: agrupamento com K-Means (exploratório) e classificação com Random Forest.

## Dados

- **Fonte:** Google Cluster Data 2011, versão 2 (clusterdata-2011-2), bucket público `gs://clusterdata-2011-2`, lido diretamente por HTTPS nos notebooks.
- **Tabelas usadas:** `task_events` (eventos e atributos de submissão) e `task_usage` (consumo de recursos em janelas de até 5 minutos).
- **Amostra:** jobs com `job_id % 10 == 0` (cerca de 10% dos jobs), preservando os 29 dias do trace.
- Os dados brutos e intermediários **não** ficam no repositório; são gravados no Google Drive (`MyDrive/artigo_predicao`).

## Estrutura

```
notebooks/
  01_eventos_tarefas.ipynb        rótulo, amostra e unidade de análise (primeira tentativa)
  02_uso_tarefas.ipynb            horizonte de 10 min e agregação do consumo por tarefa
  03_analise_exploratoria.ipynb   análise exploratória (seção 5 do artigo)
  04_agrupamento.ipynb            K-Means (a incluir)
  05_classificacao.ipynb          Random Forest e comparação (a incluir)
figuras/                          figuras usadas no artigo
requirements.txt
```

## Como reproduzir

1. Abra os notebooks no Google Colab, na ordem numérica.
2. Cada notebook monta o Google Drive e usa a pasta `MyDrive/artigo_predicao`.
3. As etapas de leitura dos 500 arquivos de cada tabela gravam um parquet por arquivo; se a sessão cair, basta rodar a célula de novo que os arquivos já processados são pulados.

## Principais decisões de método

| Decisão | Justificativa |
|---|---|
| Rótulo FAIL x FINISH | EVICT, KILL e LOST refletem remoção pelo escalonador, encerramento pelo usuário ou perda do registro, não o desfecho da execução |
| Primeira tentativa como unidade | exigir um único evento terminal excluía 99,4% das tarefas com falha; usar todas as tentativas deixava 82% das falhas vindas de tarefas com mais de 100 ressubmissões |
| Horizonte de 10 min por timestamp | só entram medições encerradas até 10 min antes do evento terminal |
| Divisão cronológica | treino nos dias 1 a 23 (validação nos dias 20 a 23) e teste nos dias 24 a 29 |
| Dias 2 e 10 mantidos no treino | na primeira tentativa, a taxa de falha desses dias fica abaixo da média |

## Resultados intermediários (amostra de 10%)

| Etapa | Tarefas | Falhas |
|---|---|---|
| Primeira tentativa (FAIL ou FINISH) | 1.821.654 | 100.730 (5,53%) |
| Base de modelagem (com janela válida antes do horizonte) | 724.349 | 46.045 (6,36%) |
| Teste (dias 24 a 29) | 134.744 | 6,73% |

## Referência principal

SANTOS, J. C. **Aprendizado de máquina aplicado à predição de falhas de software na computação em nuvem**. 2023. Dissertação (Mestrado em Informática Aplicada) – Universidade Federal Rural de Pernambuco, Recife, 2023.
