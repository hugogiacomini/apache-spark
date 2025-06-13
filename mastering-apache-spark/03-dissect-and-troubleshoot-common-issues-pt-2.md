# Dissecando e Solucionando Problemas Comuns no Apache Spark (Parte 2)

Este documento aprofunda os principais desafios de performance e troubleshooting em aplicações Apache Spark, detalhando conceitos como Shuffle, Spill, Skew, particionamento, gerenciamento de memória, serialização e técnicas avançadas como Salting. O conteúdo é baseado em demonstrações práticas, discussões e perguntas frequentes, visando capacitar profissionais a identificar, analisar e otimizar pipelines Spark em ambientes distribuídos.

---

## 1. Shuffle: O que é, como monitorar e otimizar

### O que é Shuffle?

Shuffle é o processo de redistribuição de dados entre partições, geralmente necessário em operações como `join`, `groupBy`, `distinct`, entre outras. É uma das operações mais custosas do Spark, pois envolve leitura e escrita em disco e transferência de dados pela rede.

### Como monitorar Shuffle?

- **Spark UI**: Permite visualizar os estágios (`stages`) e tarefas (`tasks`) que envolvem shuffle, além de métricas detalhadas como tempo de execução, bytes lidos/escritos, quantidade de tarefas, etc.
- **Spark Manager / Stage Metrics**: Ferramenta que coleta métricas detalhadas de execução, facilitando a análise de performance sem depender apenas do Spark UI.

### Otimizando Shuffle

- **Configuração de Partições**: O parâmetro `spark.sql.shuffle.partitions` controla o número de partições geradas após um shuffle. O padrão é 200, mas pode ser ajustado conforme o tamanho do dataset e recursos do cluster.

```python
spark.conf.set("spark.sql.shuffle.partitions", 100)
```

- **Buffer de Shuffle**: O parâmetro `spark.reducer.maxSizeInFlight` define o tamanho do buffer de transferência de dados durante o shuffle. Aumentar esse valor pode melhorar a performance em operações de shuffle intensivo.

```python
spark.conf.set("spark.reducer.maxSizeInFlight", "96m")
```

- **Broadcast Hash Join (BHJ)**: Forçar o uso de BHJ em joins onde uma das tabelas é pequena pode eliminar o shuffle, reduzindo drasticamente o tempo de execução.

```python
from pyspark.sql.functions import broadcast
df_joined = df_large.join(broadcast(df_small), on="id")
```

- **Adaptive Query Execution (AQE)**: O Spark pode ajustar dinamicamente o número de partições e estratégias de join.

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

---

## 2. Spill: O que é, causas e mitigação

### O que é Spill?

Spill ocorre quando não há memória suficiente para processar dados intermediários, forçando o Spark a escrever parte desses dados em disco. Isso acontece principalmente durante operações de shuffle, sort e join.

### Estratégias de Mitigação

- **Aumentar a memória dos executores** (`spark.executor.memory`).

```python
spark.conf.set("spark.executor.memory", "4g")
```

- **Reduzir o skew** (ver seção Skew).

- **Ajustar o tamanho das partições** (`spark.sql.files.maxPartitionBytes`).

```python
spark.conf.set("spark.sql.files.maxPartitionBytes", 64 * 1024 * 1024)  # 64MB
```

- **Revisar uso de cache/persist**: Remover persistências desnecessárias libera memória.

```python
df.unpersist()
```

- **Processar dados em chunks**: Dividir o processamento em partes menores.

```python
for year in [2021, 2022]:
    df_year = df.filter(df.ano == year)
    # processa df_year
```

---

## 3. Gerenciamento de Memória no Spark

### Ajustes e Otimizações

- **Aumentar memória**: Quando possível, é a solução mais simples para spills recorrentes.

```python
spark.conf.set("spark.executor.memory", "8g")
```

- **Ajustar frações de memória**: Parâmetros como `spark.memory.fraction` e `spark.memory.storageFraction` permitem customizar a alocação entre execution e storage.

```python
spark.conf.set("spark.memory.fraction", 0.7)
spark.conf.set("spark.memory.storageFraction", 0.3)
```

- **Entender o impacto do cache**: Cache/persist consome storage memory; uso excessivo pode causar spill.

```python
df.persist()
# ... uso do df ...
df.unpersist()
```

---

## 4. Skew: Identificação e Técnicas de Mitigação

### Técnicas de Mitigação

- **Isolar dados skewed**: Processar separadamente partições ou chaves que concentram muitos registros.

```python
skewed_keys = [1, 2]
df_skewed = df.filter(df.id.isin(skewed_keys))
df_normal = df.filter(~df.id.isin(skewed_keys))
# Processar separadamente
```

- **Broadcast Hash Join**: Quando possível, elimina shuffle e reduz o impacto do skew.

```python
df_joined = df_large.join(broadcast(df_small), on="id")
```

- **Salting**: Técnica avançada que distribui artificialmente chaves skewed em múltiplos buckets, balanceando a carga. (Veja seção Salting)

- **Ajustar particionamento**: Usar `repartition`, `coalesce` e ajustar `maxPartitionBytes` para tentar uniformizar as partições.

```python
df_repart = df.repartition(100, "id")
```

---

## 5. Salting: Técnica Avançada para Resolver Skew

### Exemplo de Salting em PySpark

```python
from pyspark.sql import functions as F

num_buckets = 3
df1_salted = df1.withColumn("salt", F.floor(F.rand() * num_buckets))

df2_expanded = df2.crossJoin(
    spark.range(0, num_buckets).withColumnRenamed("id", "salt")
)

joined = df1_salted.join(
    df2_expanded,
    on=["id", "salt"],
    how="inner"
)
```

---

## 6. Storage: Cache e Persistência

### Exemplo de uso de cache e persist

```python
df_cached = df.cache()
df_cached.count()  # materializa o cache
# ... uso do df_cached ...
df_cached.unpersist()
```

```python
from pyspark import StorageLevel
df_persisted = df.persist(StorageLevel.MEMORY_AND_DISK)
df_persisted.count()
df_persisted.unpersist()
```

---

## 7. Serialização: Impacto e Boas Práticas

### Ativar Kryo Serializer

```python
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

### Preferir funções nativas e Pandas UDFs

```python
# Função nativa
df = df.withColumn("nova_col", F.col("valor") * 2)

# Pandas UDF
import pandas as pd
from pyspark.sql.functions import pandas_udf

@pandas_udf("double")
def multiplica_por_dois(v: pd.Series) -> pd.Series:
    return v * 2

df = df.withColumn("nova_col", multiplica_por_dois("valor"))
```

---

## 8. Dicas Gerais de Troubleshooting

- **Sempre monitore as métricas**: Use Spark UI, Spark Manager e logs para identificar gargalos.
- **Ajuste o particionamento**: Quantidade e tamanho das partições impactam diretamente a performance.

```python
df = df.repartition(50)
```

- **Evite operações custosas**: `orderBy`, `distinct`, `explode`, `crossJoin` podem gerar shuffle e spill.

```python
# Exemplo de uso controlado de orderBy
df = df.orderBy("coluna").limit(100)
```

- **Reveja lógica de negócio**: Filtros e joins mal planejados são fontes comuns de problemas.

```python
df = df.filter(df.status == "ativo")
```

- **Documente e salve métricas**: Armazene relatórios de execução para análise histórica e RCA.

---

## 9. Perguntas Frequentes

- **Como salvar métricas em arquivo?**  
    Use as funções do Spark Manager para exportar relatórios em CSV ou outros formatos.

```python
df_metrics.write.csv("/caminho/para/metrics.csv")
```

- **Como ver configurações da sessão Spark?**  

```python
spark.conf.getAll()
```

- **Como identificar problemas de rede?**  
    Observe tempos de espera elevados, tarefas reiniciadas e métricas de shuffle remoto.

- **Como processar dados em chunks?**  

```python
for chunk in range(0, 100, 10):
    df_chunk = df.filter((df.id >= chunk) & (df.id < chunk + 10))
    # processa df_chunk
```

---

## 10. Conclusão

Dominar troubleshooting e performance tuning no Apache Spark exige conhecimento profundo dos mecanismos internos da engine, análise criteriosa das métricas e aplicação de técnicas avançadas como salting, particionamento inteligente e uso eficiente de memória. Com as ferramentas e práticas apresentadas, é possível identificar rapidamente gargalos, propor soluções e garantir pipelines de dados escaláveis e eficientes.

---

**Continue estudando, revisite este material sempre que necessário e pratique em ambientes reais para consolidar o aprendizado!**
