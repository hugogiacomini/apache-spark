# Melhores Práticas de Programação com Apache Spark

## Introdução

Olá, pessoal! Neste documento, vamos abordar em detalhes as melhores práticas para desenvolvimento de aplicações com Apache Spark, tanto em batch quanto em streaming. O objetivo é fornecer uma visão abrangente sobre como estruturar, modularizar, testar e monitorar aplicações Spark, além de apresentar exemplos práticos em PySpark para cada conceito.

---

## Agenda

- Fundamentos essenciais para engenheiros de dados
- Arquitetura do Spark e seus pilares
- Modularização e padrões de projeto para ETL
- Gerenciamento de configuração
- Logging e monitoramento
- Testes e validação de dados
- Práticas para processamento em batch e streaming
- Exemplos práticos em PySpark

---

## 1. Fundamentos para Engenheiros de Dados

Antes de mergulhar no Spark, é fundamental dominar conceitos de engenharia de dados, como:

- O que é um Data Lake
- Arquiteturas de dados (bronze, silver, gold)
- Formatos de arquivo (Parquet, ORC, Avro, Delta, Iceberg)
- Princípios de particionamento e distribuição de dados

**Dica:** Se sentir necessidade, busque treinamentos focados em fundamentos antes de avançar para tópicos avançados de Spark.

---

## 2. Arquitetura do Spark

Compreender a arquitetura do Spark é essencial para escrever código eficiente:

- **Driver:** Coordena a execução da aplicação.
- **Executors:** Executam as tarefas distribuídas.
- **Cluster Manager:** Gerencia recursos do cluster.
- **Tasks e Partições:** Cada partição de dados é processada por uma task.

**Exemplo de inicialização de sessão Spark em PySpark:**

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MinhaAplicacaoSpark") \
    .getOrCreate()
```

---

## 3. Modularização e Padrões de Projeto

### Separation of Concerns (Separação de Responsabilidades)

Divida sua aplicação em módulos claros:

- **Ingestion:** Leitura dos dados
- **Transformation:** Transformações e lógica de negócio
- **Output:** Escrita dos dados

**Exemplo de estrutura de projeto:**

```
src/
  ingestion.py
  transformation.py
  output.py
  main.py
config/
  config.json
utils/
  spark_utils.py
```

### Exemplo prático

#### ingestion.py

```python
def read_data(spark, file_path, file_format="parquet"):
    return spark.read.format(file_format).load(file_path)
```

#### transformation.py

```python
def filter_active_business(df):
    return df.filter(df["is_active"] == True)

def calculate_average_rating(df):
    return df.groupBy("business_id").avg("rating")
```

#### output.py

```python
def write_data(df, output_path, file_format="parquet"):
    df.write.format(file_format).mode("overwrite").save(output_path)
```

#### main.py

```python
import json
from pyspark.sql import SparkSession
from ingestion import read_data
from transformation import filter_active_business, calculate_average_rating
from output import write_data

with open("config/config.json") as f:
    config = json.load(f)

spark = SparkSession.builder.appName("ETLApp").getOrCreate()

df = read_data(spark, config["input_path"])
df_active = filter_active_business(df)
df_avg = calculate_average_rating(df_active)
write_data(df_avg, config["output_path"])
```

---

## 4. Gerenciamento de Configuração

Nunca armazene configurações sensíveis ou específicas do ambiente no código. Use arquivos externos (JSON, YAML, etc).

**Exemplo de config.json:**

```json
{
  "input_path": "s3://meu-bucket/dados/input/",
  "output_path": "s3://meu-bucket/dados/output/"
}
```

---

## 5. Logging e Monitoramento

Adicione logs em pontos críticos do seu pipeline para facilitar troubleshooting.

**Exemplo:**

```python
import logging

logging.basicConfig(level=logging.INFO)
logging.info("Iniciando leitura dos dados...")
```

---

## 6. Testes e Validação de Dados

### Testes Unitários com PySpark

Utilize frameworks como `unittest`, `pytest`, ou o módulo nativo `pyspark.testing`.

**Exemplo de teste:**

```python
from pyspark.sql import SparkSession
import unittest

class TestTransformations(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.spark = SparkSession.builder.master("local[1]").appName("Test").getOrCreate()

    def test_filter_active_business(self):
        data = [("A", True), ("B", False)]
        df = self.spark.createDataFrame(data, ["business_id", "is_active"])
        from transformation import filter_active_business
        result = filter_active_business(df)
        self.assertEqual(result.count(), 1)

if __name__ == "__main__":
    unittest.main()
```

### Validação de Dados

Utilize bibliotecas como [YData Profiling](https://github.com/ydataai/ydata-profiling), [PyDeequ](https://github.com/awslabs/deequ), [Chispa](https://github.com/MrPowers/chispa) e [Queen](https://github.com/queensql/queen).

**Exemplo de profiling com YData Profiling:**

```python
import pandas as pd
from ydata_profiling import ProfileReport

df = spark.read.parquet("meu_arquivo.parquet").toPandas()
profile = ProfileReport(df, title="Relatório de Profiling")
profile.to_file("report.html")
```

---

## 7. Práticas para Processamento em Batch

- **Evite formatos ineficientes** (JSON, CSV) para ingestão. Prefira Parquet, ORC, Delta, Iceberg.
- **Otimize particionamento**: Use `repartition` ou `coalesce` para evitar small files.
- **Evite shuffles desnecessários**: Prefira funções nativas do Spark ao invés de UDFs em Python.
- **Utilize broadcast joins** para datasets pequenos.

**Exemplo de broadcast join:**

```python
from pyspark.sql.functions import broadcast

df1 = spark.read.parquet("grande.parquet")
df2 = spark.read.parquet("pequeno.parquet")
result = df1.join(broadcast(df2), "chave")
```

---

## 8. Práticas para Processamento em Streaming

### Estrutura Básica do Structured Streaming

```python
stream_df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "meu_topico") \
    .load()

from pyspark.sql.functions import from_json, col

schema = ... # Defina seu schema aqui
parsed_df = stream_df.select(from_json(col("value").cast("string"), schema).alias("data"))

query = parsed_df.writeStream \
    .format("parquet") \
    .option("path", "/caminho/saida") \
    .option("checkpointLocation", "/caminho/checkpoint") \
    .start()

query.awaitTermination()
```

### Janelas de Tempo (Windows)

- **Tumbling Window:** Janela fixa, sem sobreposição.
- **Sliding Window:** Janela móvel, com sobreposição.
- **Session Window:** Baseada em sessões de atividade.

**Exemplo de Tumbling Window:**

```python
from pyspark.sql.functions import window

agg_df = parsed_df.groupBy(
    window(col("event_time"), "10 minutes")
).count()
```

---

## 9. Padrões de Projeto Recomendados

- **Factory:** Criação de objetos complexos.
- **Builder:** Construção passo a passo de objetos.
- **Decorator:** Adiciona funcionalidades sem alterar a estrutura principal.

**Referência:** [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)

---

## 10. Dicas Finais

- **Documente seu código** e utilize ferramentas de IA para gerar documentação e testes.
- **Gatekeeper:** Tenha revisores criteriosos para garantir qualidade do código.
- **Evite grandes pull requests**; prefira pequenas entregas incrementais.
- **Acompanhe a evolução do PySpark Testing** para testes nativos.

---

## 11. Recursos Úteis

- [PySpark Documentation](https://spark.apache.org/docs/latest/api/python/)
- [YData Profiling](https://github.com/ydataai/ydata-profiling)
- [PyDeequ](https://github.com/awslabs/deequ)
- [Chispa](https://github.com/MrPowers/chispa)
- [Queen](https://github.com/queensql/queen)
- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)

---

## Conclusão

Seguindo essas práticas, você estará apto a desenvolver pipelines Spark robustos, escaláveis e fáceis de manter, tanto em batch quanto em streaming. Pratique, revise e evolua constantemente seu conhecimento!

