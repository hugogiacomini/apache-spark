# Último Dia: Arquiteturas, Design Patterns e Melhores Práticas com Apache Spark

Bem-vindo ao último dia do nosso treinamento! Neste documento, vamos consolidar os principais conceitos, padrões de arquitetura e melhores práticas para trabalhar com Apache Spark em ambientes de Big Data. O objetivo é que você saia daqui com uma visão 360º, capaz de arquitetar soluções robustas, eficientes e alinhadas com as necessidades do negócio.

---

## 1. **Ciclo de Vida de uma Aplicação Spark**

O Spark é um motor de processamento em memória, altamente eficiente, projetado para trabalhar principalmente com Data Lakes. Seu ciclo de vida envolve:

- **Ingestão de dados**: Receber dados de múltiplas fontes (JDBC, APIs, arquivos, etc).
- **Processamento**: Aplicar regras de negócio, transformações e análises.
- **Entrega**: Salvar resultados em Data Lakes, Data Warehouses, bancos relacionais, ou sistemas de visualização.

> **Dica:** O Spark não é a melhor ferramenta para ingestão massiva de dados brutos. Use ferramentas especializadas para ingestão e utilize o Spark para processamento e análise.

---

## 2. **Data Lake, Lakehouse e Arquitetura Medallion**

### **Data Lake**

- Armazena dados brutos, de múltiplos formatos e fontes.
- Permite flexibilidade, mas pode virar um "data swamp" se não houver governança.

### **Lakehouse**

- Combina a flexibilidade do Data Lake com as garantias ACID e esquema do Data Warehouse.
- Exemplos: Delta Lake, Apache Iceberg, Hudi.

### **Arquitetura Medallion (Bronze, Silver, Gold)**

- **Bronze**: Dados brutos, ingestão direta.
- **Silver**: Dados limpos, integrados e estruturados.
- **Gold**: Dados prontos para consumo analítico e de negócio.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("MedallionArchitecture").getOrCreate()

# Bronze: ingestão de dados brutos
df_bronze = spark.read.json("s3://datalake/bronze/dados.json")

# Silver: limpeza e integração
df_silver = df_bronze.filter("status = 'ativo'").dropDuplicates(["id"])

# Gold: agregações para negócio
df_gold = df_silver.groupBy("categoria").count()

df_gold.write.format("delta").mode("overwrite").save("s3://datalake/gold/relatorio_categoria")
```

---

## 3. **Plano de Execução no Spark**

O Spark transforma seu código em um plano de execução otimizado, passando por:

- **Plano Lógico**: Representação da consulta.
- **Catalyst Optimizer**: Otimiza o plano lógico, gera múltiplos planos físicos e escolhe o de menor custo.
- **Plano Físico**: Instruções reais para execução distribuída.
- **RDD**: Execução em baixo nível, geralmente em Scala/Java.

### **Visualizando o Plano de Execução**

```python
df_gold.explain(True)
```

Saída típica:
```
== Physical Plan ==
*(2) HashAggregate(keys=[categoria#12], functions=[count(1)])
+- Exchange hashpartitioning(categoria#12, 200)
    +- *(1) HashAggregate(keys=[categoria#12], functions=[partial_count(1)])
        +- *(1) Project [...]
            +- *(1) Filter (isnotnull(status#10) && (status#10 = ativo))
                +- FileScan json [...]
```

- **Exchange**: Indica shuffle (movimentação de dados entre nós).
- **WholeStageCodegen**: Otimização para execução eficiente.
- **ScanParquet/ScanDelta**: Leitura eficiente de arquivos colunares.

---

## 4. **Operadores e Otimizações**

- **Filter Pushdown**: Filtros aplicados antes de carregar dados para memória.
- **Shuffle/Exchange**: Operações caras (joins, repartition, orderBy).
- **Broadcast Hash Join**: Join eficiente quando uma tabela é pequena.

```python
from pyspark.sql.functions import broadcast

# Exemplo de Broadcast Join
df1 = spark.read.parquet("s3://datalake/gold/tabela_grande")
df2 = spark.read.parquet("s3://datalake/gold/tabela_pequena")

df_join = df1.join(broadcast(df2), "chave")
```

---

## 5. **Adaptive Query Execution (AQE)**

A partir do Spark 3.0, o AQE permite otimizações dinâmicas durante a execução, como ajuste automático de partições e escolha de joins.

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
```

---

## 6. **Orquestração de Pipelines**

O Spark precisa ser orquestrado para compor pipelines de dados complexos. Ferramentas comuns:

- **Airflow** (open source, multi-cloud)
- **Azure Data Factory** (Azure)
- **AWS Glue** (AWS)
- **Google Cloud Data Fusion** (GCP)

Exemplo de DAG no Airflow para Spark:

```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from datetime import datetime

with DAG("pipeline_spark", start_date=datetime(2024, 1, 1), schedule_interval="@daily") as dag:
     spark_job = SparkSubmitOperator(
          task_id="processa_dados",
          application="/caminho/para/job.py",
          conn_id="spark_default"
     )
```

---

## 7. **Arquiteturas Lambda, Kappa e Layered**

### **Lambda**

- Separa processamento batch e streaming.
- Mais complexa, mas útil para cenários legados.

### **Kappa**

- Unifica ingestão e processamento via eventos (Kafka).
- Simplifica a arquitetura, ideal para sistemas event-driven.

### **Layered**

- Arquitetura em camadas: ingestão, exploração, processamento, entrega.
- Agnóstica de nuvem e tecnologia.

---

## 8. **Comparação de Nuvens e Custos**

- **Google Cloud** costuma ser mais barato para pipelines de Big Data, devido à arquitetura containerizada.
- **AWS** e **Azure** oferecem maior diversidade de serviços, mas podem ter custos mais altos.
- Planeje e compare custos antes de definir a stack.

---

## 9. **Stack Moderna de Big Data**

- **Processamento**: Spark, Trino, Dremio
- **Mensageria/Eventos**: Kafka, Pulsar
- **Orquestração**: Airflow, ArgoCD
- **Visualização**: Metabase, Superset
- **Monitoramento**: Prometheus, Grafana
- **Armazenamento**: Data Lake (S3, GCS, ADLS), Delta Lake, Iceberg

---

## 10. **Habilidades Essenciais para Engenheiros de Dados**

- **Fundamentos**: ETL, ELT, Data Warehouse, modelagem de dados, SQL, Python.
- **Bancos de Dados**: Relacionais (Postgres, MySQL), NoSQL (MongoDB, Cassandra, Redis).
- **Sistemas Distribuídos**: Hadoop, Spark, Kafka, Airflow.
- **Cloud**: Escolha uma (AWS, Azure, GCP) e aprofunde-se.
- **Soft Skills**: Comunicação, resolução de problemas, colaboração, adaptabilidade.

---

## 11. **Certificações Recomendadas**

- **Google Professional Data Engineer**
- **AWS Data Analytics Specialty**
- **Azure Data Engineer Associate**
- **Databricks Certified Associate Developer for Apache Spark**

---

## 12. **Dicas Finais**

- Foque em fundamentos e pratique diariamente.
- Use o Spark UI e o History Server para analisar e otimizar jobs.
- Prefira formatos colunares (Parquet, Delta) para performance.
- Evite shuffles desnecessários, otimize joins e partições.
- Orquestre pipelines com Airflow ou ferramentas nativas da nuvem.
- Mantenha-se atualizado com as tendências do mercado e open source.
- Invista em soft skills e networking.

---

## 13. **Exemplo Completo: Pipeline Medallion com PySpark**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count

spark = SparkSession.builder.appName("PipelineMedallion").getOrCreate()

# Bronze: ingestão
df_bronze = spark.read.json("s3://datalake/bronze/dados.json")
df_bronze.write.format("delta").mode("overwrite").save("s3://datalake/bronze/delta")

# Silver: limpeza
df_silver = df_bronze.filter(col("ativo") == True).dropDuplicates(["id"])
df_silver.write.format("delta").mode("overwrite").save("s3://datalake/silver/delta")

# Gold: agregação
df_gold = df_silver.groupBy("categoria").agg(count("*").alias("total"))
df_gold.write.format("delta").mode("overwrite").save("s3://datalake/gold/delta")
```

---

## 14. **Referências e Próximos Passos**

- [Documentação Oficial do Apache Spark](https://spark.apache.org/docs/latest/)
- [Delta Lake](https://delta.io/)
- [Databricks Academy](https://academy.databricks.com/)
- [Google Cloud Data Engineer Learning Path](https://cloud.google.com/certification/data-engineer)
- [Airflow Docs](https://airflow.apache.org/docs/)

---

> **Lembre-se:** O sucesso depende mais da sua constância, curiosidade e desejo de aprender do que do medo de errar. Pratique, experimente, colabore e evolua sempre!

