# Lakehouse como Fundação e Pronto para Produção

## Introdução

Bem-vindos ao último dia do treinamento Mastering Apache Spark! Hoje vamos abordar o componente final da nossa jornada: **Lakehouse** como fundação para ambientes de produção, com foco em deploys no **Kubernetes**. Este documento detalha conceitos, arquitetura, melhores práticas e exemplos práticos com PySpark para que você possa aplicar o que há de mais moderno em engenharia de dados.

---

## 1. Conceitos Fundamentais

### 1.1 Data Lake vs Lakehouse

- **Data Lake**: Armazena dados brutos, geralmente em formatos como Parquet, ORC ou Avro, sem estrutura rígida.
- **Lakehouse**: Une o melhor do Data Lake (baixo custo, flexibilidade) com o Data Warehouse (transações ACID, governança, performance).

#### Exemplo PySpark: Leitura de dados Parquet

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("LakehouseExample").getOrCreate()
df = spark.read.parquet("s3a://meu-bucket/datalake/dados/")
df.show()
```

---

### 1.2 Arquitetura Moderna: Fair House

- **Fair House**: Evolução do Lakehouse, permitindo escolha de engine de processamento e local de armazenamento, com formatos abertos e interoperáveis.
- **Open Table Formats (OTF)**: Delta Lake, Apache Iceberg, Apache Hudi.

---

## 2. Camadas da Arquitetura Lakehouse

1. **Storage**: S3, ADLS, GCS, MinIO, etc.
2. **File Format**: Parquet, ORC, Avro.
3. **Table Format**: Delta Lake, Iceberg, Hudi.
4. **Storage Engine**: Gerencia manutenção, atualização e organização dos dados.
5. **Catálogo**: Unity Catalog, Hive Metastore, Glue, etc.
6. **Query Engines**: Spark, Trino, Presto, Dremio, etc.

---

## 3. Delta Lake vs Iceberg

### 3.1 Principais Features

- **Transações ACID**: Garantem que operações de escrita e leitura sejam atômicas, consistentes, isoladas e duráveis, prevenindo corrupção de dados e garantindo integridade mesmo em falhas.
- **Time Travel**: Permite consultar versões anteriores dos dados, facilitando auditoria, recuperação e análises históricas.
- **Schema Evolution**: Suporta alterações no esquema das tabelas (adição, remoção ou alteração de colunas) sem interromper operações existentes.
- **Change Data Feed (CDF)**: Disponibiliza um fluxo de alterações (inserções, atualizações, deleções) realizadas nos dados, útil para integrações e replicações incrementais.
- **Hidden Partitioning (Iceberg)**: Gerencia partições de forma transparente, eliminando a necessidade de manipulação manual e reduzindo erros em queries.
- **Uniform Table Format (Delta Uniform)**: Proporciona interoperabilidade entre diferentes engines e ferramentas, padronizando o formato de tabelas Delta para facilitar integrações.

#### Exemplo PySpark: Criando uma tabela Delta

```python
df.write.format("delta").mode("overwrite").save("s3a://meu-bucket/lakehouse/tabela_delta")
```

#### Time Travel

```python
df = spark.read.format("delta").option("versionAsOf", 3).load("s3a://meu-bucket/lakehouse/tabela_delta")
```

#### Change Data Feed

```python
df_cdf = spark.read.format("delta") \
    .option("readChangeData", "true") \
    .option("startingVersion", 5) \
    .load("s3a://meu-bucket/lakehouse/tabela_delta")
```

---

## 4. Arquitetura Well-Architected Data Lakehouse

- **Bronze**: Dados brutos.
- **Silver**: Dados limpos e enriquecidos.
- **Gold**: Dados prontos para consumo analítico.

#### Exemplo PySpark: Pipeline Bronze → Silver → Gold

```python
# Bronze
bronze_df = spark.read.json("s3a://meu-bucket/raw/")
bronze_df.write.format("delta").save("s3a://meu-bucket/bronze/")

# Silver
silver_df = bronze_df.filter("status = 'ativo'")
silver_df.write.format("delta").save("s3a://meu-bucket/silver/")

# Gold
gold_df = silver_df.groupBy("categoria").count()
gold_df.write.format("delta").save("s3a://meu-bucket/gold/")
```

---

## 5. Otimizações e Features Avançadas

### 5.1 Data Skipping

O Delta Lake utiliza estatísticas de arquivos para acelerar queries, lendo apenas arquivos relevantes.

### 5.2 Z-Ordering

Organiza fisicamente os dados para acelerar queries em colunas específicas.

```python
from delta.tables import DeltaTable

deltaTable = DeltaTable.forPath(spark, "s3a://meu-bucket/lakehouse/tabela_delta")
deltaTable.optimize().executeZOrderBy("coluna_importante")
```
### 5.3 Liquid Clustering

Organização adaptativa dos dados baseada em padrões de acesso, balanceando arquivos automaticamente.

#### Exemplo PySpark: Aplicando Liquid Clustering com Delta Lake

> **Observação:** O Liquid Clustering está disponível a partir do Delta Lake 3.0+ (Databricks). Para ambientes open source, verifique a disponibilidade do recurso.

```python
from delta.tables import DeltaTable

deltaTable = DeltaTable.forPath(spark, "s3a://meu-bucket/lakehouse/tabela_delta")

# Configura o clustering automático (liquid clustering)
deltaTable.optimize().executeLiquidClusterBy("coluna_clusterizacao")
```

- Substitua `"coluna_clusterizacao"` pela coluna que representa o critério de clusterização (ex: `user_id`, `data_evento`).
- O Delta Lake irá reorganizar os arquivos de acordo com o padrão de acesso, otimizando consultas futuras.
- Consulte a documentação oficial para detalhes sobre parâmetros avançados e monitoramento do clustering.


---

## 6. Deploy de Spark no Kubernetes

### 6.1 Por que usar Kubernetes?

- **Escalabilidade**: O Kubernetes permite escalar automaticamente os recursos de Spark (drivers e executors) conforme a demanda, otimizando o uso de infraestrutura e garantindo performance mesmo em grandes volumes de dados.

- **Isolamento**: Cada job Spark roda em pods isolados, evitando conflitos de dependências e facilitando a execução simultânea de múltiplos pipelines sem interferência.

- **Redução de custos**: Com o gerenciamento eficiente de recursos e o suporte a clusters elásticos, o Kubernetes possibilita alocação sob demanda, reduzindo custos com infraestrutura ociosa.

- **Portabilidade**: Aplicações Spark empacotadas em containers podem ser executadas em qualquer ambiente compatível com Kubernetes (on-premises ou cloud), facilitando migração e padronização de ambientes.

### 6.2 Componentes

### 6.2 Componentes

- **Pods**: São as menores unidades de execução no Kubernetes. No contexto do Spark, cada job é composto por um pod driver (responsável por coordenar a execução) e múltiplos pods executors (que processam os dados em paralelo). Cada pod possui seu próprio ambiente isolado, facilitando o gerenciamento de recursos e dependências.

- **Namespaces**: Permitem a segmentação lógica do cluster Kubernetes, isolando recursos, permissões e políticas entre diferentes equipes, projetos ou ambientes (ex: desenvolvimento, homologação, produção). Isso garante organização, segurança e controle de acesso refinado.

- **Spark Operator**: É um controlador customizado para Kubernetes que automatiza o deploy, monitoramento e gerenciamento do ciclo de vida de aplicações Spark. Ele interpreta recursos do tipo `SparkApplication` (YAML), cria e gerencia os pods necessários, lida com falhas, escalonamento e integrações com outros serviços do cluster.

### 6.3 Exemplo: Deploy de Job PySpark no Kubernetes

#### Dockerfile para aplicação PySpark

```dockerfile
FROM bitnami/spark:3.5
COPY app /app
WORKDIR /app
```

#### YAML de SparkApplication

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: etl-job
  namespace: processing
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: meu-registry/etl-job:latest
  mainApplicationFile: local:///app/main.py
  sparkVersion: "3.5.0"
  driver:
    cores: 1
    memory: "1g"
  executor:
    cores: 1
    memory: "1g"
    instances: 2
  restartPolicy:
    type: OnFailure
    onFailureRetries: 3
```

---

## 7. Desenvolvimento Ágil: Scaffold

Utilize o [Skaffold](https://skaffold.dev/) para automatizar build e deploy contínuo de imagens e jobs Spark no Kubernetes.

---

## 8. Integração com Trino/Presto

Após processar e gravar dados no Lakehouse, utilize engines como Trino para consultas SQL rápidas e escaláveis.

---

## 9. Recomendações Finais

- **Desacople configurações sensíveis** (ex: credenciais) do código.
- **Automatize deploys** com CI/CD.
- **Monitore e otimize** jobs e recursos.
- **Aproveite features avançadas** do Delta/Iceberg para performance e governança.

---

## 10. Referências e Próximos Passos

- [Documentação Delta Lake](https://docs.delta.io/latest/)
- [Documentação Apache Iceberg](https://iceberg.apache.org/)
- [Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html)
- [Skaffold](https://skaffold.dev/)

---

## 11. Conclusão

O Lakehouse representa o estado da arte em arquitetura de dados, unindo flexibilidade, performance e governança. Com Spark, Delta/Iceberg e Kubernetes, você está pronto para construir pipelines robustos, escaláveis e prontos para produção.

---

**Bons estudos e sucesso na sua jornada em Engenharia de Dados!**
