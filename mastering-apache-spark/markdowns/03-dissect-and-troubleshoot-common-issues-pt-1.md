# Dissect and Troubleshoot Common Issues - Part 1

## Introdução

Neste terceiro dia do treinamento de Mastering Apache Spark, vamos aprofundar a análise e resolução dos principais problemas enfrentados em Spark, conhecidos como os "5S": Spill, Skill, Shuffle, Storage e Serialization. O objetivo é entender como cada um desses problemas afeta o processamento distribuído e como aplicar as melhores práticas para solucioná-los.

---

## Revisão das Demos Anteriores: Coalesce e Repartition

### Conceitos Fundamentais

- **Coalesce**: Operação utilizada para reduzir o número de partições de um DataFrame. É considerada uma *Narrow Transformation* porque não envolve shuffle entre executores; apenas reduz partições existentes no mesmo executor. Não é possível aumentar o número de partições com `coalesce`.

    ```python
    # Exemplo de uso do coalesce
    df = spark.read.parquet("dados.parquet")
    df_coalesced = df.coalesce(4)
    print(df_coalesced.rdd.getNumPartitions())
    ```

- **Repartition**: Permite tanto aumentar quanto diminuir o número de partições. Envolve shuffle completo dos dados entre executores, sendo uma *Wide Transformation*.

    ```python
    # Exemplo de uso do repartition
    df_repartitioned = df.repartition(8)
    print(df_repartitioned.rdd.getNumPartitions())
    ```

### Demonstração Prática

- **Tempo de execução**: Ao executar um `count` em um DataFrame com 16 partições, o tempo foi de 7 segundos. Após reparticionar para 24 partições, o tempo caiu para 4 segundos, mostrando o custo do shuffle.

    ```python
    import time
    df = spark.range(0, 10000000).repartition(16)
    start = time.time()
    df.count()
    print("Tempo com 16 partições:", time.time() - start)

    df = df.repartition(24)
    start = time.time()
    df.count()
    print("Tempo com 24 partições:", time.time() - start)
    ```

- **Limitações do Coalesce**: Ao tentar aumentar de 16 para 24 partições usando `coalesce`, o número de partições permanece em 16.

    ```python
    df = df.coalesce(24)
    print(df.rdd.getNumPartitions())  # Continua 16
    ```

- **Cálculo ideal de partições**:

    ```python
    # Exemplo de cálculo
    num_nos = 3
    cores_por_no = 2
    fator_paralelismo = 4
    num_particoes = num_nos * cores_por_no * fator_paralelismo
    print("Número ideal de partições:", num_particoes)
    ```

---

## Perguntas Frequentes e Boas Práticas

- **Ordem das operações (Select/Filter)**:

    ```python
    # Ambas as ordens são otimizadas pelo Catalyst
    df.select("col1", "col2").filter("col1 > 10")
    df.filter("col1 > 10").select("col1", "col2")
    ```

- **Leitura de múltiplas tabelas**:

    ```python
    # Evite collect em grandes volumes
    for table in ["tabela1", "tabela2"]:
            df = spark.read.table(table)
            df.show(5)
    ```

- **Configuração mínima de ambiente**:

    ```python
    # Exemplo de configuração local
    spark = SparkSession.builder \
            .master("local[4]") \
            .config("spark.driver.memory", "6g") \
            .getOrCreate()
    ```

- **Uso de Coalesce para arquivos pequenos**:

    ```python
    # Consolidando arquivos pequenos
    df = spark.read.parquet("input/")
    df.coalesce(1).write.mode("overwrite").parquet("output/")
    ```

- **Ajuste de partições via configuração**:

    ```python
    spark.conf.set("spark.sql.files.maxPartitionBytes", 128 * 1024 * 1024)  # 128MB por partição
    ```

---

## Diferença entre Partições Lógicas e Físicas

- **Partições do Spark**:

    ```python
    df = spark.read.parquet("dados.parquet")
    print("Partições lógicas:", df.rdd.getNumPartitions())
    ```

- **Partições no Storage**:

    ```python
    # Salvando dados particionados por coluna
    df.write.partitionBy("ano", "mes").parquet("output/")
    ```

---

## Cache e Persistência

### Conceitos

- **Cache**:

    ```python
    df = spark.read.parquet("dados.parquet")
    df.cache()
    df.count()  # Materializa o cache
    ```

- **Persist**:

    ```python
    from pyspark.storagelevel import StorageLevel
    df.persist(StorageLevel.MEMORY_AND_DISK)
    df.count()
    ```

### Demonstração

- **Sem cache / Com cache**:

    ```python
    import time
    df = spark.range(0, 100000000)
    start = time.time()
    df.filter("id % 2 == 0").count()
    print("Sem cache:", time.time() - start)

    df.cache()
    start = time.time()
    df.filter("id % 2 == 0").count()
    print("Com cache:", time.time() - start)
    ```

### Detalhes Técnicos

- **Onde o cache é armazenado?**: Nos executores.
- **Diferença para Broadcast**:

    ```python
    from pyspark.sql.functions import broadcast
    small_df = spark.read.parquet("dimensao.parquet")
    large_df = spark.read.parquet("fato.parquet")
    result = large_df.join(broadcast(small_df), "chave")
    ```

- **Ações para ativar cache**:

    ```python
    df.cache()
    df.count()  # Materializa o cache
    ```

- **Escopo do cache**: Válido apenas na sessão Spark atual.

---

## Broadcast Join

### Conceito

- **Broadcast Join**:

    ```python
    from pyspark.sql.functions import broadcast
    small_df = spark.read.parquet("dimensao.parquet")
    large_df = spark.read.parquet("fato.parquet")
    result = large_df.join(broadcast(small_df), "chave")
    ```

- **Configuração**:

    ```python
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 20 * 1024 * 1024)  # 20MB
    ```

- **Uso prático**:

    ```python
    result = large_df.join(broadcast(small_df), "chave")
    ```

### Demonstração

- **Sem broadcast / Com broadcast**:

    ```python
    # Sem broadcast
    result = large_df.join(small_df, "chave")
    # Com broadcast
    result = large_df.join(broadcast(small_df), "chave")
    ```

---

## Ordem dos Joins e Otimização

### Conceito

- **Ordem dos Joins**:

    ```python
    transacoes = spark.read.parquet("transacoes.parquet")
    paises = spark.read.parquet("paises.parquet")
    clientes = spark.read.parquet("clientes.parquet")

    # Join eficiente: broadcast nas tabelas pequenas
    result = transacoes.join(broadcast(paises), "pais_id") \
                                         .join(broadcast(clientes), "cliente_id")
    ```

### Adaptive Query Execution (AQE)

- **Otimização automática**:

    ```python
    spark.conf.set("spark.sql.adaptive.enabled", "true")
    ```

---

## Shuffle: O Coração do Spark

### Conceito

- **Shuffle**:

    ```python
    df = spark.read.parquet("dados.parquet")
    df.groupBy("chave").count().show()
    ```

### Detalhamento do Processo

```python
# Exemplo de shuffle em um groupBy
df = spark.range(0, 1000000).withColumn("grupo", (df.id % 10))
df.groupBy("grupo").count().show()
```

### Estratégias para Reduzir Shuffle

- **Ajuste do número de partições**:

    ```python
    spark.conf.set("spark.sql.shuffle.partitions", 100)
    ```

- **Uso de formatos eficientes**:

    ```python
    df.write.parquet("output/")
    ```

- **Repartition e Coalesce**:

    ```python
    df = df.repartition(10)
    df = df.coalesce(2)
    ```

### External Shuffle Service

- **Configuração (exemplo)**:

    ```python
    spark.conf.set("spark.shuffle.service.enabled", "true")
    ```

---

## Conclusão

Compreender profundamente os conceitos de partições, cache, persistência, broadcast, ordem dos joins e, principalmente, o funcionamento do shuffle é fundamental para escrever aplicações Spark eficientes e escaláveis. O domínio desses tópicos permite identificar gargalos, otimizar recursos e garantir a performance em ambientes distribuídos.

---

**Próximos passos**: Nas próximas sessões, serão abordadas melhores práticas de estruturação de código, design patterns, e a implementação de um caso de uso completo, aplicando todos os conceitos discutidos.

