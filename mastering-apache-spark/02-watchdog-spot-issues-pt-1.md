# Watchdog Spot Issues - Part 1

Neste documento, vamos detalhar a estrutura do repositório utilizado no treinamento de Apache Spark, explicando cada pasta, como configurar o ambiente local, executar as demonstrações e entender os principais conceitos de performance tuning, patterns e antipatterns no Spark.

---

## 1. Estrutura do Repositório

O repositório está organizado para facilitar a reprodução das demonstrações e o estudo dos conceitos apresentados. As principais pastas são:

- **build**: Scripts e arquivos para criar o ambiente local do Spark usando Docker.
- **logs**: Armazena os logs gerados pelas aplicações Spark.
- **metrics**: Guarda métricas coletadas, especialmente pelo Spark Measure.
- **app**: Aplicações de exemplo que escrevem dados em storage (ex: MinIO).
- **src**: Scripts principais do treinamento, incluindo datasets de exemplo.

### Como utilizar

1. **Clonar o repositório** e navegar até a pasta desejada.
2. **Criar um arquivo `.env`** com os caminhos locais para storage, logs, metrics, etc.
3. **Executar o build das imagens Docker** para Spark e Spark History Server.
4. **Subir o ambiente com Docker Compose**:
    ```bash
    docker-compose up -d
    ```
5. **Verificar os containers em execução**:
    ```bash
    docker ps
    ```
    Você verá containers para master, workers e history server.

### Requisitos mínimos

- 2 workers (pode ajustar para 3, se tiver mais recursos)
- 4 cores de CPU e 6GB de RAM recomendados para ambiente local

---

## 2. Acesso e Mapeamento de Pastas

O Docker Compose mapeia as pastas locais para dentro dos containers, permitindo que tudo que for colocado em `src`, `storage`, `logs`, etc., seja automaticamente acessível pelos jobs Spark.

---

## 3. Aplicação de Exemplo

A aplicação de exemplo (`app`) gera arquivos em formatos Parquet ou JSON dentro do storage (MinIO). Para rodar:

- Configure o `.env` com as credenciais do MinIO.
- Escolha o formato de saída (Parquet ou JSON).
- Execute a aplicação para gerar dados reais para as demonstrações.

**Exemplo PySpark: Gerando dados de exemplo**

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("ExemploApp").getOrCreate()

# Gerar DataFrame de exemplo
data = [("Alice", 34), ("Bob", 45), ("Carol", 29)]
df = spark.createDataFrame(data, ["nome", "idade"])

# Salvar em Parquet
df.write.mode("overwrite").parquet("/storage/exemplo.parquet")

# Salvar em JSON
df.write.mode("overwrite").json("/storage/exemplo.json")
```

---

## 4. Datasets Utilizados

Os principais datasets utilizados são:

- **Yelp Dataset**: Dados reais e volumosos para simular cenários de Big Data.
- **TLC (New York Trip Record Data)**: Outro dataset grande, ideal para testes de performance.

**Exemplo PySpark: Leitura de dataset**

```python
# Leitura de um dataset Parquet
df = spark.read.parquet("/storage/yelp_dataset.parquet")
df.printSchema()
df.show(5)
```

---

## 5. Fundamentos do Spark

### Revisão dos Conceitos

- **Arquitetura Spark**: Driver, executors, ciclo de vida da aplicação, uso da JVM, otimizações recentes.
- **Gerenciamento de recursos**: FIFO, FAIR scheduling, integração com Databricks, Dataproc, EMR, Azure.
- **RDDs, DataFrames e planos de execução**: Como o Spark processa dados internamente.

**Exemplo PySpark: Operações com DataFrame**

```python
# Filtrar e agrupar dados
df_filtered = df.filter(df["idade"] > 30)
df_grouped = df_filtered.groupBy("idade").count()
df_grouped.show()
```

**Exemplo PySpark: Visualizando plano de execução**

```python
df_grouped.explain()
```

---

## 6. Monitoramento e Diagnóstico de Performance

### Ferramentas Disponíveis

- **Spark UI**: Interface nativa para monitoramento de jobs.
- **Spark Listener & Metrics**: APIs para coleta de métricas detalhadas.
- **Spark Measure**: Wrapper que facilita a coleta e análise de métricas de performance.
- **EFK Stack (Elasticsearch, Fluentd, Kibana)**: Para centralização e análise de logs.
- **Prometheus & Grafana**: Dashboards para métricas em tempo real.

#### Recomendação

O **Spark Measure** é altamente recomendado pela facilidade de uso e riqueza de informações. Basta adicionar no início e fim do código:

```python
from sparkmeasure import StageMetrics
stagemetrics = StageMetrics(spark)
stagemetrics.begin()
# ... seu código Spark ...
stagemetrics.end()
stagemetrics.print_report()
```

---

## 7. Patterns e Antipatterns no Spark

### Por que estudar patterns e antipatterns?

- **70% dos problemas** podem ser evitados com boas práticas.
- **30% dos casos** exigem análise detalhada e tuning fino.

### Principais problemas enfrentados

- **Small Files Problem**: Muitos arquivos pequenos prejudicam performance.
- **Skew**: Distribuição desigual dos dados entre as partições.
- **Limitações de hardware**: Storage lento, pouca memória, etc.
- **Complexidade crescente dos jobs**: Mudanças no código podem degradar performance.

### Diagnóstico

- **Baseline**: Sempre tenha um tempo de execução de referência.
- **Ferramentas**: Use Spark UI, Spark Measure, logs e métricas para identificar gargalos.

**Exemplo PySpark: Reparticionamento para evitar small files**

```python
# Reparticionar antes de salvar para evitar small files
df.repartition(4).write.mode("overwrite").parquet("/storage/saida_parquet")
```

**Exemplo PySpark: Identificando skew**

```python
# Verificando distribuição de dados por partição
from pyspark.sql.functions import spark_partition_id

df.withColumn("partition_id", spark_partition_id()).groupBy("partition_id").count().show()
```

---

## 8. Formatos de Arquivo: Comparação Detalhada

### JSON

- **Prós**: Fácil integração com microserviços.
- **Contras**: Não é colunar, parsing caro, sem compressão nativa.

### ORC

- **Prós**: Formato colunar, ótimo para integração com Hive, compressão eficiente.
- **Contras**: Menos utilizado fora do ecossistema Hive.

### Avro

- **Prós**: Ótimo para streaming, serialização eficiente.
- **Contras**: Baseado em linhas, menos eficiente para analytics em Spark.

### Parquet

- **Prós**: Formato colunar, compressão eficiente, integração nativa com Spark, ideal para analytics.
- **Contras**: Poucos.

#### Tabela Comparativa

| Formato | Tipo         | Compressão | Pushdown | Caso de Uso Ideal           |
|---------|--------------|------------|----------|-----------------------------|
| JSON    | Texto        | Não        | Não      | Microserviços, integração   |
| ORC     | Colunar      | Sim        | Sim      | Hive, grandes volumes       |
| Avro    | Linha        | Sim        | Não      | Streaming, serialização     |
| Parquet | Colunar      | Sim        | Sim      | Analytics, Data Lake, Delta |

**Exemplo PySpark: Leitura e escrita em diferentes formatos**

```python
# Leitura de JSON
df_json = spark.read.json("/storage/exemplo.json")

# Escrita em ORC
df_json.write.mode("overwrite").orc("/storage/exemplo.orc")

# Escrita em Avro (necessário pacote extra)
df_json.write.format("avro").mode("overwrite").save("/storage/exemplo.avro")
```

---

## 9. Demonstração Prática: JSON vs Parquet

### Cenário

- 188 arquivos JSON (~34.9 KB cada) e 188 arquivos Parquet (~18 KB cada) com os mesmos dados.
- Leitura dos arquivos via Spark, análise do número de partições e tempo de processamento.

**Exemplo PySpark: Comparando leitura de JSON e Parquet**

```python
import time

# Leitura JSON
start = time.time()
df_json = spark.read.json("/storage/json_files/")
print("JSON count:", df_json.count())
print("JSON partitions:", df_json.rdd.getNumPartitions())
print("Tempo JSON:", time.time() - start)

# Leitura Parquet
start = time.time()
df_parquet = spark.read.parquet("/storage/parquet_files/")
print("Parquet count:", df_parquet.count())
print("Parquet partitions:", df_parquet.rdd.getNumPartitions())
print("Tempo Parquet:", time.time() - start)
```

### Resultados

- **JSON**: 6.4 MB lidos, 6 partições, tempo maior para listar arquivos.
- **Parquet**: 3.5 MB lidos, mesma quantidade de arquivos, quase 50% de economia em storage e I/O.

#### Conclusão

- Apenas mudar o formato de arquivo de JSON para Parquet reduz drasticamente o volume de dados lidos e melhora a performance.
- Em grandes volumes, o ganho é ainda mais significativo.

---

## 10. Recomendações Finais

- Sempre converta dados brutos (JSON, CSV) para formatos colunares (Parquet, ORC) antes de processar.
- Utilize ferramentas de monitoramento e métricas para identificar e corrigir gargalos.
- Siga patterns recomendados e evite antipatterns comuns para garantir performance e escalabilidade.

---

## 11. Próximos Passos

- Explorar em detalhes os principais antipatterns (Small Files, Skew, etc.).
- Realizar demonstrações práticas de cada caso.
- Aprender a utilizar o Spark Measure e outras ferramentas para análise avançada.

---

**Dica:** Aproveite o conteúdo, faça perguntas e pratique com os datasets sugeridos para consolidar o aprendizado!

