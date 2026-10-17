# Predição de Falhas em Tarefas em Ambientes de Computação em Nuvem

Código do artigo desenvolvido na disciplina Análise de Dados e Aprendizado de Máquina (PPGIA/UFRPE).

O trabalho prevê, com **10 minutos de antecedência**, se a primeira tentativa de execução de uma tarefa no Google Cluster Data 2011 terminará em falha (FAIL) ou em conclusão normal (FINISH). São usadas duas etapas de aprendizado de máquina sobre a mesma base: uma análise não supervisionada, que compara **K-Means** e **DBSCAN**, e uma classificação supervisionada, que compara **Random Forest** e **XGBoost**.

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
  04_agrupamento.ipynb            K-Means e DBSCAN, taxa de falha por grupo e sensibilidade
  05_classificacao.ipynb          Random Forest e XGBoost (a incluir)
figuras/                          figuras usadas no artigo (inclui as do agrupamento)
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
| Divisão cronológica | treino nos dias 1 a 19, validação nos dias 20 a 23 (hiperparâmetros e limiar de decisão) e teste nos dias 24 a 29; o modelo final é reajustado nos dias 1 a 23 (treino e validação), como em Santos (2023); as estatísticas exploratórias também usam os dias 1 a 23, para não consultar o teste |
| Divisão por job | cada job vai por inteiro para o conjunto do dia de sua primeira tarefa, o que elimina a sobreposição de jobs entre treino, validação e teste (485.308, 104.308 e 134.733 tarefas; 10.110 tarefas mudam de conjunto em relação à divisão pelo dia de cada tarefa) |
| Peso por job | cada tarefa pesa o inverso do tamanho do seu job, para que um job com milhares de tarefas não pese como milhares de falhas independentes |
| Jobs dominantes | as comparações entre classes são reportadas com e sem os três jobs de maior número de falhas nos dias 1 a 23 (análise de sensibilidade; nenhum registro é removido da base de modelagem) |
| Dias 2 e 10 mantidos no treino | na primeira tentativa, a taxa de falha desses dias fica abaixo da média |
| Atributos | conjunto de Santos (2023), sem os tempos de disco (não registrados a partir do 15º dia) |
| Valores ausentes | CPI e MAI são os únicos atributos com ausência (18,0% e 21,0% das tarefas dos dias 1 a 23); são imputados pela mediana do treino, com indicador de ausência, porque a taxa de falha entre as tarefas com ausência é menor |
| Modelo principal sem `tempo_obs` e `n_janelas` | carregam a duração total da tarefa por causa do corte ancorado no término; entram apenas em uma variante de comparação |
| Algoritmos | não supervisionado: K-Means (k pelo cotovelo e pela silhueta) e DBSCAN; supervisionado: Random Forest (`class_weight='balanced'`) e XGBoost (`scale_pos_weight`) |

## Resultados intermediários (amostra de 10%)

| Etapa | Tarefas | Falhas |
|---|---|---|
| Primeira tentativa (FAIL ou FINISH) | 1.821.654 | 100.730 (5,53%) |
| Base de modelagem (com janela válida antes do horizonte) | 724.349 | 46.045 (6,36%) |
| Treino (jobs iniciados nos dias 1 a 19) | 485.308 | 30.164 (6,22%) |
| Validação (jobs iniciados nos dias 20 a 23) | 104.308 | 6.823 (6,54%) |
| Teste (jobs iniciados nos dias 24 a 29) | 134.733 | 9.058 (6,72%) |

Na análise exploratória (notebook 03), os 10 jobs com mais falhas concentram 75,6% das falhas da base de modelagem, e o maior deles (job 6337545460, dia 11) responde sozinho por 42,5%. O job 6449919400, do dia 25, responde por 73% das falhas do teste.

## Agrupamento (notebook 04)

- K-Means: k = 2 pela silhueta média (0,681); o cotovelo, traçado a partir de k = 1, indica k = 3; k = 3 entra na análise de sensibilidade.
- DBSCAN: minPts = 22, amostra de 60.560 tarefas (limite de 50 por job). O cotovelo indica ε = 1,6239, que excede o limite de 4 GB de memória do notebook; foi adotado ε = 1,0, com sensibilidade em ε = 0,5, 0,7 e 1,3 (vetores idênticos reunidos em um ponto com peso).
- A taxa de falha por grupo e o escore de atipicidade (ROC-AUC e PR-AUC) são avaliados com e sem o job com mais falhas de cada conjunto; as saídas (`agrupamento_*.csv`) ficam no Google Drive, em `MyDrive/artigo_predicao/saida`, e não no repositório.

## Referência principal

SANTOS, J. C. **Aprendizado de máquina aplicado à predição de falhas de software na computação em nuvem**. 2023. Dissertação (Mestrado em Informática Aplicada) – Universidade Federal Rural de Pernambuco, Recife, 2023.
