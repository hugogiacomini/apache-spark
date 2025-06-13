# Engenharia de Dados com Apache Spark: ETL Batch, Data Lake, Lakehouse e Delta Lake

Este documento detalha conceitos fundamentais de engenharia de dados moderna, com foco em processamento em batch utilizando Apache Spark, Data Lake, Lakehouse e Delta Lake. O conteúdo foi elaborado para fornecer uma compreensão profunda dos conceitos, arquitetura, melhores práticas e exemplos práticos utilizando PySpark.

---

## 1. Introdução ao Kubernetes e Containers

O **Kubernetes** é um orquestrador de containers que revolucionou a forma como aplicações são implantadas, escaladas e mantidas. Ele garante alta disponibilidade, auto-recuperação (self-healing) e abstrai a complexidade de infraestrutura.

### Conceitos-Chave

- **Container**: Unidade leve e portátil para empacotar aplicações e suas dependências.
- **Docker**: Plataforma popular para criação e execução de containers.
- **Kubernetes**: Orquestrador que gerencia múltiplos containers em clusters de máquinas físicas ou virtuais.

### Exemplo de Arquitetura

```
+-------------------+
|   Usuário Final   |
+--------+----------+
         |
         v
+--------+----------+
|    Serviço        | <--- Abstração de acesso
+--------+----------+
         |
         v
+-------------------+
|   Pods (Containers)|
+--------+----------+
         |
         v
+-------------------+
|   Nodes (VMs/Físico)|
+-------------------+
```

---

## 2. Evolução da Infraestrutura: Físico, VM, Containers

- **Máquinas Físicas**: Servidores dedicados, alta capacidade, mas pouco flexíveis.
- **Máquinas Virtuais (VMs)**: Várias VMs em um mesmo hardware, cada uma com seu próprio SO.
- **Containers**: Compartilham o kernel do SO, são mais leves e rápidos para iniciar.

**Problema resolvido pelos containers:** Isolamento de dependências, portabilidade e facilidade de replicação.

---

## 3. Data Lake: Conceito, Desafios e Práticas

### O que é Data Lake?

Um **Data Lake** é um repositório centralizado que permite armazenar dados estruturados, semi-estruturados e não estruturados em sua forma bruta (raw).

#### Características

- **Escalabilidade**: Armazena grandes volumes de dados.
- **Flexibilidade**: Suporta múltiplos formatos (CSV, JSON, Parquet, etc).
- **Democratização**: Dados disponíveis para múltiplos times e ferramentas.

#### Desafios

- **Data Swamp**: Sem governança, o Data Lake pode virar um "pântano" de dados desorganizados.
- **Governança e Qualidade**: Necessidade de catalogação, versionamento e controle de acesso.

#### Boas Práticas

- **Zonas de Dados**: Organize o Data Lake em zonas (Landing, Processing, Curated).
- **Documentação**: Registre decisões arquiteturais e processos de ingestão.

---

## 4. Lakehouse e Delta Lake: O Futuro do Armazenamento Analítico

### Limitações do Data Lake

- Falta de transações ACID.
- Dificuldade de leitura eficiente em grandes volumes.
- Ausência de versionamento e time travel.

### O que é Lakehouse?

**Lakehouse** é um conceito que une o melhor do Data Lake (flexibilidade, baixo custo) com o Data Warehouse (transações, performance, governança).

### Delta Lake

- **Formato open source** criado pela Databricks.
- Adiciona transações ACID, time travel, schema enforcement e evolução ao Data Lake.
- Baseado em arquivos Parquet + camada de metadados (Delta Log).

#### Principais Benefícios

- **Transações ACID**: Segurança e confiabilidade.
- **Time Travel**: Consulta de versões anteriores dos dados.
- **Schema Evolution**: Evolução de esquema sem dor.
- **Performance**: Data skipping, leitura eficiente.

---

## 5. Arquitetura Medallion (Bronze, Silver, Gold)

A arquitetura Medallion organiza o processamento de dados em camadas:

- **Bronze**: Dados crus, ingestão direta das fontes.
- **Silver**: Dados limpos e enriquecidos.
- **Gold**: Dados prontos para consumo analítico e BI.

```
Fonte de Dados --> Bronze --> Silver --> Gold --> BI/Analytics
```

---

## 6. PySpark: Exemplos Práticos de ETL Batch

### 6.1. Leitura de Dados

#### Lendo arquivos JSON do Data Lake

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("ETL-Batch").getOrCreate()

# Lendo arquivos JSON
df_bronze = spark.read.json("abfss://landing@storageaccount.dfs.core.windows.net/bank/*.json")
df_bronze.printSchema()
df_bronze.show(5)
```

### 6.2. Escrita em Parquet (Pré-Lakehouse)

```python
# Escrevendo em Parquet (formato otimizado para Big Data)
df_bronze.write.mode("overwrite").parquet("abfss://processing@storageaccount.dfs.core.windows.net/bank/")
```

### 6.3. Escrita em Delta Lake (Lakehouse)

```python
# Escrevendo em Delta Lake (Lakehouse)
df_bronze.write.format("delta").mode("overwrite").save("abfss://delta@storageaccount.dfs.core.windows.net/batch/bronze/bank/")
```

### 6.4. Leitura e Consulta em Delta Lake

```python
# Lendo dados Delta
df_delta = spark.read.format("delta").load("abfss://delta@storageaccount.dfs.core.windows.net/batch/bronze/bank/")
df_delta.createOrReplaceTempView("bank_bronze")

# Consultando com SQL
result = spark.sql("""
    SELECT account_id, SUM(balance) as total_balance
    FROM bank_bronze
    GROUP BY account_id
""")
result.show()
```

### 6.5. Time Travel no Delta Lake

```python
# Lendo versão anterior dos dados (time travel)
df_old = spark.read.format("delta").option("versionAsOf", 0).load("abfss://delta@storageaccount.dfs.core.windows.net/batch/bronze/bank/")
df_old.show(5)
```

### 6.6. Evolução de Esquema

```python
# Permitindo evolução de esquema ao escrever
df_bronze.write.format("delta").option("mergeSchema", "true").mode("append").save("abfss://delta@storageaccount.dfs.core.windows.net/batch/bronze/bank/")
```

---

## 7. Boas Práticas e Dicas Profissionais

- **Use Delta Lake sempre que possível** para garantir confiabilidade e performance.
- **Evite pandas para grandes volumes**: utilize DataFrames do PySpark.
- **Documente decisões arquiteturais** e processos de ingestão.
- **Implemente governança**: catálogos, versionamento, controle de acesso.
- **Otimize partições**: utilize `repartition` e `coalesce` conforme necessário.
- **Monitore e audite**: aproveite os metadados do Delta Lake para rastreabilidade.

---

## 8. Referências

- [Documentação Oficial do Apache Spark](https://spark.apache.org/docs/latest/)
- [Delta Lake Documentation](https://docs.delta.io/latest/)
- [Databricks Documentation](https://docs.databricks.com/)
- [Azure Data Lake Storage Gen2](https://docs.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
- [PySpark API Reference](https://spark.apache.org/docs/latest/api/python/)

---

## Conclusão

O domínio de Data Lake, Lakehouse e Delta Lake, aliado ao uso correto do Apache Spark, é fundamental para engenheiros de dados modernos. A arquitetura Medallion, o uso de Delta Lake e as boas práticas apresentadas garantem pipelines escaláveis, confiáveis e prontos para os desafios de Big Data.

