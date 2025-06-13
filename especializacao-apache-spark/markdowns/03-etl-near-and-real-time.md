# Processamento de Dados em Near Real-Time e Streaming com Apache Spark

Este documento detalha o processo de construção de pipelines de dados em batch e streaming utilizando Apache Spark, Delta Lake e Kafka, com exemplos práticos em PySpark. O objetivo é fornecer uma visão clara e aplicada sobre como arquitetar soluções de dados modernas, desde a ingestão até a modelagem de tabelas de domínio, passando por melhores práticas e desafios reais.

---

## 1. Introdução ao Delta Lake e Data Lakehouse

O **Delta Lake** é uma camada de armazenamento que traz transações ACID, versionamento e schema enforcement para o Data Lake, permitindo a construção de arquiteturas Lakehouse. Ele não fornece conexão Direct Query, sendo necessário um Data Warehouse (DW) para certas funcionalidades analíticas.

### Camadas Bronze, Silver e Gold

- **Bronze**: Dados crus, sem tratamento, equivalente ao "staging" do DW. Permite retenção infinita e versionamento.
- **Silver**: Dados refinados, tratados e enriquecidos, prontos para análises e integrações.
- **Gold**: Dados prontos para consumo analítico, relatórios e dashboards.

---

## 2. Pipeline Batch: Da Bronze à Silver

### 2.1. Leitura de Dados Delta

```python
# Leitura de tabelas Delta no Spark
df_user = spark.read.format("delta").load("/mnt/datalake/bronze/users")
df_ssn = spark.read.format("delta").load("/mnt/datalake/bronze/ssn")
```

### 2.2. Exploração de Estruturas Complexas

O Delta Lake permite visualizar metadados e lineage dos dados, além de facilitar o acesso a estruturas aninhadas (structs):

```python
# Seleção de campos, incluindo structs
from pyspark.sql.functions import col

df_user.select(
    col("user_id"),
    col("employment.title").alias("user_title"),
    col("employment.keySkills").alias("user_skills")
).show()
```

### 2.3. Flatten de Estruturas JSON

Para análises tabulares, é comum "flattenar" estruturas aninhadas:

```python
df_flat = df_user.select(
    col("uuid").alias("user_uid"),
    col("id").alias("user_id"),
    col("first_name"),
    col("last_name"),
    col("date_of_birth").cast("timestamp").alias("user_birth_date"),
    col("gender"),
    col("employment.title").alias("user_title"),
    col("employment.keySkills").alias("user_skills")
).distinct()
```

### 2.4. Registro de Views Temporárias

```python
df_flat.createOrReplaceTempView("enhanced_users")
df_ssn.createOrReplaceTempView("enhanced_ssn")
```

### 2.5. Joins e Escrita na Silver

```python
# Join em SQL
df_joined = spark.sql("""
SELECT u.*, s.ssn
FROM enhanced_users u
LEFT JOIN enhanced_ssn s ON u.user_id = s.user_id
""")

# Escrita na Silver
df_joined.write.format("delta").mode("overwrite").save("/mnt/datalake/silver/users")
```

---

## 3. Modelagem de Domain Tables

Ao migrar da Bronze para Silver, recomenda-se criar **tabelas de domínio** (domain tables), que agregam informações relevantes para o negócio, facilitando análises e garantindo atomicidade e integridade dos dados.

### Exemplo de Estrutura de Domain Table

```python
from pyspark.sql.functions import lit

df_domain = df_joined.withColumn("source_system", lit("mongo")) \
    .withColumn("event_time", current_timestamp()) \
    .withColumn("is_active", lit(True))
```

---

## 4. Processamento de Eventos e Streaming

### 4.1. Conceitos Fundamentais

- **Event Stream**: Fluxo contínuo e ordenado de eventos (unbounded).
- **Event Stream Processing (ESP)**: Processamento eficiente de eventos em tempo real.
- **Garantias de Entrega**:
  - *At most once*: Pode perder eventos.
  - *At least once*: Pode duplicar eventos.
  - *Exactly once*: Sem perda ou duplicidade (Kafka + Spark).

### 4.2. Arquitetura com Kafka e Spark

O Kafka atua como um barramento central de eventos, desacoplando produtores e consumidores, e permitindo ingestão e processamento em tempo real.

---

## 5. Structured Streaming com Spark

### 5.1. Leitura de Dados em Streaming do Data Lake

```python
from pyspark.sql.types import StructType, StructField, StringType

# Definindo o schema
user_schema = StructType([
    StructField("user_id", StringType(), True),
    StructField("first_name", StringType(), True),
    # ... outros campos
])

# Leitura em streaming
df_stream = spark.readStream \
    .schema(user_schema) \
    .json("/mnt/datalake/landing/users/")
```

#### Observação

O processamento em streaming de arquivos no Data Lake é **near real-time**, pois depende de listagens periódicas de diretórios, o que pode gerar latência em grandes volumes.

### 5.2. Escrita em Delta Lake com Checkpoint

```python
df_stream.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/mnt/datalake/checkpoints/bronze_users") \
    .start("/mnt/datalake/bronze/users")
```

### 5.3. Enriquecimento em Streaming

```python
df_bronze = spark.readStream.format("delta").load("/mnt/datalake/bronze/users")

df_silver = df_bronze.withColumn("full_name", concat_ws(" ", "first_name", "last_name"))

df_silver.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/mnt/datalake/checkpoints/silver_users") \
    .start("/mnt/datalake/silver/users")
```

---

## 6. Otimizando Streaming: Databricks Cloud Files

O **Cloud Files** do Databricks otimiza o processamento de arquivos em streaming, utilizando listagem paralela ou notificações de eventos (Event Grid/SNS), reduzindo a latência.

```python
df_cloud = spark.readStream \
    .format("cloudFiles") \
    .option("cloudFiles.format", "json") \
    .schema(user_schema) \
    .load("/mnt/datalake/landing/users/")
```

---

## 7. Ingestão de Dados do Kafka

### 7.1. Leitura de Streaming do Kafka

```python
df_kafka = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker:9092") \
    .option("subscribe", "music_data_json") \
    .option("startingOffsets", "latest") \
    .load()

from pyspark.sql.functions import from_json, col

df_parsed = df_kafka.select(
    from_json(col("value").cast("string"), user_schema).alias("data")
).select("data.*")
```

### 7.2. Escrita em Delta Lake

```python
df_parsed.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/mnt/datalake/checkpoints/bronze_music") \
    .start("/mnt/datalake/bronze/music")
```

---

## 8. Boas Práticas e Observações

- **Separação de Notebooks**: Recomenda-se separar pipelines de bronze, silver e gold em notebooks distintos.
- **Checkpoints**: Essenciais para garantir exactly-once e recuperação de falhas.
- **Domain Tables**: Facilitam a modelagem e manutenção de dados de negócio.
- **Cloud Files**: Use para otimizar ingestão em Databricks.
- **Kafka + Spark**: Arquitetura recomendada para streaming real-time e escalável.

---

## 9. Novidades: Project Lightspeed

O Spark está evoluindo com o **Project Lightspeed**, que trará:
- Suporte a múltiplos joins stateful em streaming.
- Checkpoints assíncronos.
- Melhorias de performance (30-50%+).

---

## 10. Conclusão

A combinação de Delta Lake, Spark Structured Streaming e Kafka permite construir pipelines de dados robustos, escaláveis e prontos para o futuro do processamento de dados em tempo real. Pratique os exemplos, adapte para seu contexto e explore as possibilidades!

---

**Dica**: Sempre consulte a [documentação oficial do Spark](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) e do [Delta Lake](https://docs.delta.io/latest/delta-intro.html) para detalhes atualizados.

