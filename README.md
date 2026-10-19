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
  05_classificacao.ipynb          Random Forest e XGBoost, variantes de atributos e SHAP
  06_intervalos_bootstrap.ipynb   intervalos de confiança por bootstrap de jobs
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

## Notebooks

Todos leem e gravam em `MyDrive/artigo_predicao` (dados intermediários em `saida/`, que não faz parte do repositório).

**01. Eventos de tarefa** (`01_eventos_tarefas.ipynb`)
- Faz: lê `task_events` em lotes, define o rótulo FAIL x FINISH e compara três unidades de análise; adota a primeira tentativa de cada tarefa.
- Saída: `saida/primeira_tentativa.parquet` (1.821.654 tarefas, 100.730 falhas).

**02. Uso de recursos** (`02_uso_tarefas.ipynb`)
- Faz: lê `task_usage`, aplica o horizonte de 10 min por timestamp e agrega o consumo por tarefa.
- Saída: `saida/base_modelagem.parquet` (724.349 tarefas, 46.045 falhas).

**03. Análise exploratória** (`03_analise_exploratoria.ipynb`, Seção 5 do artigo)
- Faz: taxa de falha por dia, concentração de falhas por job, valores ausentes de CPI e MAI e comparação dos atributos entre as classes, com e sem os jobs dominantes. As estatísticas por dia usam o dia de cada tarefa; a divisão por job é feita no notebook 04.
- Saída: `figuras/taxa_falha_dia.png`, `figuras/concentracao_falhas_job.png` e `figuras/atributos_por_classe.png`.

**04. Agrupamento** (`04_agrupamento.ipynb`, Seção 6.1 do artigo)
- Faz: divisão por job; K-Means (k = 2 pela silhueta média de 0,681; o cotovelo, traçado a partir de k = 1, indica k = 3, que entra na sensibilidade); DBSCAN com minPts = 22 em amostra de 60.560 tarefas (limite de 50 por job). O ε do cotovelo (1,6239) excede o limite de 4 GB de memória do notebook, então foi adotado ε = 1,0, com sensibilidade em ε = 0,5, 0,7 e 1,3 (vetores idênticos reunidos em um ponto com peso). Avalia a taxa de falha por grupo e o escore de atipicidade (ROC-AUC e PR-AUC), com e sem o job com mais falhas de cada conjunto.
- Saída: `saida/base_conjuntos.parquet` e `saida/agrupamento_*.csv` (no Drive, não no repositório); figuras `kmeans_cotovelo_silhueta.png`, `kmeans_silhueta_grupos.png`, `agrupamento_pca.png` e `dbscan_kdistance.png`.

**05. Classificação** (`05_classificacao.ipynb`, Seção 6.2 do artigo)
- Faz: Random Forest (`class_weight='balanced'`) e XGBoost (`scale_pos_weight`) com peso por job; busca aleatória de hiperparâmetros na validação (6 configurações no Random Forest e 10 no XGBoost, critério PR-AUC ponderada por job); limiar de decisão que maximiza o F1 da falha na validação; reajuste nos dias 1 a 23 e avaliação no teste, com todos os jobs e sem o job dominante do teste. Variantes de atributos: principal (priority, cpu_req, ram_req, disk_req e 11 métricas de consumo), com tempo observado (`tempo_obs`, `n_janelas`), com filtro de Pearson (|r| > 0,8, calculado nos dias 1 a 19), as duas juntas e uma extensão com `scheduling_class`, hora do dia e dia da semana (atraso de escalonamento não extraído: `EXTRAIR_ATRASO = False`). SHAP (TreeExplainer) no XGBoost principal, em amostras de 2.000 tarefas do teste.
- Grava checkpoints em `saida/classificacao_ckpt/` (busca de hiperparâmetros e cada par variante x algoritmo); se a sessão cair, rodar o notebook de novo do início retoma do que já foi concluído (`RETOMAR = True`).
- Saída: `saida/classificacao_*.csv` (métricas, limiares, hiperparâmetros, busca, filtro de Pearson e SHAP), `saida/classificacao_predicoes_teste.parquet` e as figuras `classificacao_curvas_teste.png`, `shap_importancia.png` e `shap_resumo.png`.

**06. Intervalos de confiança** (`06_intervalos_bootstrap.ipynb`, Seção 6.2 do artigo)
- Faz: bootstrap de jobs (2.000 reamostragens dos 1.697 jobs do teste, limiar fixo) sobre as predições do notebook 05, com intervalos de 95% para ROC-AUC, PR-AUC, precisão, recall e F1 da falha, e diferenças pareadas entre variantes e entre algoritmos. Não reajusta modelos.
- Saída: `saida/classificacao_bootstrap.csv` e `saida/classificacao_bootstrap_diferencas.csv`.

## Referência principal

SANTOS, J. C. **Aprendizado de máquina aplicado à predição de falhas de software na computação em nuvem**. 2023. Dissertação (Mestrado em Informática Aplicada) – Universidade Federal Rural de Pernambuco, Recife, 2023.
