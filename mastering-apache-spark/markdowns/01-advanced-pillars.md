# Advanced Pillars

## Introdução

Bem-vindo ao treinamento **Mastering Apache Spark**! Este documento detalha os principais conceitos avançados e internos do Apache Spark, com explicações aprofundadas sobre arquitetura, execução distribuída, gerenciamento de recursos, paralelismo, partições, modos de agendamento, tuning de performance e melhores práticas para ambientes de produção.

---

## 1. Arquitetura do Spark

### 1.1. Componentes Principais

- **Driver**: Responsável por orquestrar a execução da aplicação Spark. Recebe o código, cria o plano lógico e físico, agenda tarefas e monitora a execução.
- **Executors**: Processos distribuídos que executam as tarefas atribuídas pelo driver. Cada executor roda em uma JVM separada.
- **Cluster Manager**: Gerencia os recursos do cluster e aloca executores. Pode ser Standalone, YARN, Mesos ou Kubernetes.

### 1.2. Fluxo de Execução

1. **Submissão**: O código é submetido ao driver.
2. **Análise**: O driver analisa o código, gera o plano lógico e físico.
3. **Divisão em Estágios e Tarefas**: O DAG Scheduler divide o trabalho em estágios e tarefas.
4. **Alocação de Recursos**: O Cluster Manager aloca recursos e inicia executores.
5. **Execução**: Os executores processam as tarefas em paralelo.
6. **Monitoramento**: O driver monitora o progresso e coleta resultados.

**Exemplo PySpark: Inicializando uma sessão Spark**

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("ExemploArquitetura") \
    .getOrCreate()
```

---

## 2. Paralelismo e Partições

### 2.1. Conceitos Fundamentais

- **Partição**: Unidade básica de paralelismo no Spark. Cada partição é processada por uma tarefa.
- **Task**: Menor unidade de execução, processa uma partição.
- **Slot**: Cada core disponível em um executor representa um slot para execução de tarefas.
- **Thread Pool**: Cada executor possui seu próprio pool de threads para gerenciar tarefas.

**Exemplo PySpark: Visualizando partições**

```python
rdd = spark.sparkContext.parallelize(range(100), numSlices=4)
print("Número de partições:", rdd.getNumPartitions())
```

### 2.2. Estratégias de Particionamento

- O tamanho padrão de partição (`spark.sql.files.maxPartitionBytes`) é 128 MB.
- O número ideal de partições é de 3 a 4 vezes o número total de cores do cluster.
- O Spark tenta alinhar o particionamento dos arquivos (especialmente Parquet) com o tamanho de partição configurado, mas o resultado depende do layout dos arquivos e metadados.

**Exemplo PySpark: Lendo arquivos com particionamento customizado**

```python
df = spark.read.option("maxPartitionBytes", 64 * 1024 * 1024).parquet("caminho/para/parquet")
print("Partições após leitura:", df.rdd.getNumPartitions())
```

### 2.3. Ajustando o Paralelismo

- **Aumentar partições**: Melhora o uso dos recursos, evita executores ociosos.
- **Reduzir partições**: Pode ser útil para evitar overhead em jobs pequenos.
- **Repartition**: Cria novo particionamento, envolve shuffle (operação custosa).
- **Coalesce**: Reduz o número de partições sem shuffle (mais eficiente para diminuir partições).

**Exemplo PySpark: repartition vs coalesce**

```python
df_repart = df.repartition(10)  # Aumenta partições (com shuffle)
df_coalesce = df.coalesce(2)    # Reduz partições (sem shuffle)
```

---

## 3. Modos de Execução e Gerenciamento de Cluster

### 3.1. Modos de Deploy

- **Client Mode**: O driver roda fora do cluster (usado em desenvolvimento).
- **Cluster Mode**: O driver roda dentro do cluster (usado em produção).

### 3.2. Cluster Managers

- **Standalone**: Gerenciador nativo do Spark, simples e eficiente para ambientes controlados.
- **YARN**: Amplo suporte em ambientes Hadoop, robusto, mas com overhead de recursos.
- **Mesos**: Menos utilizado atualmente.
- **Kubernetes**: Moderno, elástico, ótimo para ambientes de microserviços e cloud-native.

### 3.3. Scheduling Modes

- **FIFO (First In, First Out)**: Jobs são executados em ordem de chegada.
- **FAIR**: Recursos são compartilhados de forma justa entre jobs, melhor para ambientes multiusuário e workloads variados.

**Exemplo PySpark: Configurando modo FAIR**

```python
spark.conf.set("spark.scheduler.mode", "FAIR")
```

---

## 4. Internals: DAG, Jobs, Stages e Tasks

### 4.1. DAG Scheduler

- Constrói o grafo acíclico direcionado (DAG) das operações.
- Divide o processamento em **jobs** (ações), **stages** (estágios) e **tasks** (tarefas).

**Exemplo PySpark: Ações e DAG**

```python
df = spark.range(0, 1000)
df_filtered = df.filter(df.id % 2 == 0)
df_filtered.count()  # Dispara job, stages e tasks
```

### 4.2. Shuffle

- Ocorre entre estágios, geralmente em operações como `groupBy`, `join`, `distinct`.
- É a operação mais custosa do Spark, pois envolve movimentação de dados entre executores.

**Exemplo PySpark: Operação que gera shuffle**

```python
df_grouped = df.groupBy((df.id % 10).alias("grupo")).count()
```

### 4.3. Whole Stage Code Generation

- Otimização que agrupa múltiplas operações em um único bytecode, reduzindo overhead de execução na JVM.
- Quanto mais operações agrupadas, melhor a performance.

---

## 5. Tuning de Performance

### 5.1. Cálculo de Partições

- Para jobs críticos, calcule o número de partições com base no tamanho dos dados e nos recursos disponíveis.
- Exemplo: Para 1 GB de dados, com partições de 64 MB, terá 16 partições. Se o cluster tem 12 cores, haverá tarefas aguardando, aproveitando melhor os recursos.

**Exemplo PySpark: Cálculo de partições**

```python
tamanho_arquivo_mb = 1024  # 1 GB
partition_size_mb = 64
num_partitions = tamanho_arquivo_mb // partition_size_mb  # 16 partições
```

### 5.2. Configurações Relevantes

- `spark.executor.memory`: Memória por executor (mínimo recomendado: 2-4 GB).
- `spark.executor.cores`: Número de cores por executor.
- `spark.sql.shuffle.partitions`: Número de partições para operações de shuffle.
- `spark.dynamicAllocation.enabled`: Ativa alocação dinâmica de executores.

**Exemplo PySpark: Configurando recursos**

```python
spark = SparkSession.builder \
    .appName("ExemploTuning") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "4") \
    .config("spark.sql.shuffle.partitions", "16") \
    .config("spark.dynamicAllocation.enabled", "true") \
    .getOrCreate()
```

### 5.3. Estratégias de Otimização

- **Ajuste do maxPartitionBytes**: Ideal quando o tamanho dos arquivos é conhecido e fixo.
- **Uso de repartition**: Necessário quando o particionamento original não é adequado.
- **Evite JSON**: Prefira formatos colunares como Parquet ou ORC para melhor performance.
- **Compactação de Small Files**: Use rotinas de compactação para evitar muitos arquivos pequenos.

**Exemplo PySpark: Salvando em Parquet e compactando arquivos pequenos**

```python
df.write.mode("overwrite").parquet("output/parquet")
# Compactação: reparticionando antes de salvar
df.coalesce(1).write.mode("overwrite").parquet("output/compactado")
```

---

## 6. DataFrames, Datasets e RDDs

- **RDD**: API de baixo nível, flexível, mas menos otimizada. Hoje é deprecated para uso direto.
- **DataFrame**: API de alto nível, estruturada, com otimizações automáticas (Catalyst Optimizer, Tungsten).
- **Dataset**: Similar ao DataFrame, mas com tipagem forte (principalmente em Scala).

**Exemplo PySpark: RDD vs DataFrame**

```python
# RDD
rdd = spark.sparkContext.parallelize([("a", 1), ("b", 2)])
print(rdd.collect())

# DataFrame
df = spark.createDataFrame([("a", 1), ("b", 2)], ["letra", "valor"])
df.show()
```

---

## 7. Query Execution e Adaptive Query Execution (AQE)

- **Catalyst Optimizer**: Otimiza o plano lógico e físico de execução.
- **Cost-Based Optimizer (CBO)**: Usa estatísticas para escolher o melhor plano.
- **Adaptive Query Execution**: Ajusta dinamicamente o plano durante a execução, otimizando joins, repartições e tratamento de skew.

**Exemplo PySpark: Ativando AQE**

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

---

## 8. Melhores Práticas

- **Entenda o particionamento dos dados** antes de processar.
- **Ajuste o número de partições** conforme o volume de dados e recursos.
- **Evite repartition desnecessário** para não incorrer em shuffle.
- **Monitore o Spark UI** para identificar stragglers, skew e gargalos.
- **Prefira formatos colunares** e evite JSON para grandes volumes.
- **Use AQE** e mantenha o Spark atualizado para aproveitar as otimizações mais recentes.

---

## 9. Exemplos Práticos

### 9.1. Cálculo de Partições

```python
# Exemplo: Calcular número ideal de partições
tamanho_arquivo_mb = 1024  # 1 GB
partition_size_mb = 64
num_partitions = tamanho_arquivo_mb // partition_size_mb  # 16 partições
```

### 9.2. Configuração de Spark

```python
spark = SparkSession.builder \
    .appName("Exemplo") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "4") \
    .config("spark.sql.files.maxPartitionBytes", "67108864")  # 64 MB
    .getOrCreate()
```

### 9.3. Leitura, transformação e escrita de dados

```python
df = spark.read.csv("input/dados.csv", header=True, inferSchema=True)
df_filtrado = df.filter(df["coluna"] > 100)
df_filtrado.write.mode("overwrite").parquet("output/resultado")
```

### 9.4. Join com AQE ativado

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
df1 = spark.range(0, 1000000)
df2 = spark.range(0, 1000)
df_join = df1.join(df2, df1.id == df2.id, "inner")
df_join.count()
```

---

## 10. Conclusão

Dominar os pilares avançados do Spark exige compreensão profunda de arquitetura, paralelismo, particionamento, tuning e execução distribuída. Com as práticas e conceitos detalhados neste documento, você estará apto a construir pipelines de dados eficientes, escaláveis e robustos em ambientes de produção.

---

**Dica Final:** Sempre ajuste e monitore suas aplicações conforme o cenário real de dados e recursos. O Spark é poderoso, mas exige tuning contínuo para extrair o máximo de performance!