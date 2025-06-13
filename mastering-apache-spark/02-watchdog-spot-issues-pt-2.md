# Watchdog and Spot Issues - Part 2

## Introdução

Neste documento, vamos abordar em detalhes os desafios e melhores práticas relacionados ao particionamento de dados, explosão de arquivos (File Explosion), problemas de small files, skew, joins e operações de leitura e escrita no Apache Spark. O conteúdo é baseado em experiências práticas e discussões avançadas sobre o tema, com foco em ambientes modernos de processamento distribuído, como Spark, Delta Lake e Iceberg.

---

## 1. Explosão de Arquivos (File Explosion) e o Problema dos Small Files

### O que é File Explosion?

File Explosion refere-se ao fenômeno de gerar uma grande quantidade de arquivos pequenos ao particionar dados de forma inadequada. Esse problema é conhecido como "Small Files Problem" e pode causar sérios impactos de performance e custo em ambientes de Big Data.

#### Por que isso acontece?

- **Particionamento excessivo**: Ao particionar dados por colunas de alta cardinalidade (muitos valores distintos), cada valor gera uma partição/arquivo.
- **Arquivos pequenos**: O Spark trabalha melhor com arquivos de tamanho intermediário (32MB a 512MB, sendo 128MB o ideal). Arquivos muito pequenos (<32MB) aumentam o overhead de leitura e escrita.

#### Impactos

- **Performance**: Muitas requisições ao storage (ex: S3, ADLS) aumentam o tempo de leitura.
- **Custo**: Serviços de storage cobram por requisição (FETs). Muitas requisições = custo elevado.
- **Gerenciamento**: Metadados desatualizados, dificuldade de manutenção e atualização de estatísticas.

#### Exemplo Prático

- Uma tabela particionada por ID gerou 12.500 arquivos pequenos, resultando em 12.500 requisições e 54 segundos de consulta.
- Ao remover o particionamento, a mesma consulta caiu para 9 segundos e apenas 18MB transferidos.

#### Exemplo PySpark

```python
# Gerando small files ao particionar por coluna de alta cardinalidade
df.write.partitionBy("id").parquet("/caminho/saida/particionado")

# Solução: evitar particionamento excessivo e usar coalesce para reduzir arquivos
df.coalesce(10).write.parquet("/caminho/saida/otimizado")
```

---

## 2. Particionamento: Quando usar e quando evitar

### Histórico

- O particionamento surgiu como prática recomendada no Hive, para otimizar consultas em ambientes Hadoop.
- Com a evolução dos formatos (Parquet, Delta, Iceberg) e dos engines (Spark), o particionamento manual perdeu parte da sua importância.

### Recomendações Atuais

- **Evite particionar por padrão**: Só particione se a tabela tiver mais de 1TB ou se todas as consultas usarem a coluna particionada como filtro.
- **Prefira layouts inteligentes**: Use recursos como Liquid Clustering (Delta) ou Hidden Partitioning (Iceberg), que otimizam o layout automaticamente.
- **Cuidado com a cardinalidade**: Particionar por colunas de alta cardinalidade gera muitos arquivos pequenos.

#### Exemplo PySpark

```python
# Particionando apenas por colunas de baixa cardinalidade (ex: ano, mês)
df.write.partitionBy("ano", "mes").parquet("/caminho/saida/particionado")
```

### Casos Especiais

- **Hive On-Premises**: Particionar por data ainda faz sentido, mas monitore o número de partições.
- **Lakehouse (Delta/Iceberg)**: Prefira não particionar manualmente; use os recursos nativos de clustering e pruning.

---

## 3. Skew: O Desbalanceamento de Partições

### O que é skew?

Skew ocorre quando as partições de dados ficam desbalanceadas, ou seja, algumas partições têm muito mais dados que outras. Isso causa:

- **Stragglers**: Tarefas que demoram muito mais para processar.
- **Subutilização do cluster**: Alguns executores ficam ociosos esperando os stragglers terminarem.

### Como identificar e resolver

- **Identificação**: Ferramentas como SparkMeasure ajudam a detectar skew automaticamente.
- **Soluções**:
    - `repartition`: Redistribui os dados entre as partições (gera shuffle).
    - `coalesce`: Reduz o número de partições sem shuffle (menos agressivo).
    - **Salting**: Técnica avançada para balancear dados com alta concentração em certos valores.

#### Exemplo PySpark

```python
# Identificando skew visualmente
df.groupBy("chave_skew").count().orderBy("count", ascending=False).show()

# Solução simples: reparticionar
df = df.repartition(20, "chave_skew")

# Salting para skew extremo
from pyspark.sql.functions import rand, concat, col

df_salted = df.withColumn("chave_skew_salt", concat(col("chave_skew"), (rand()*10).cast("int")))
df_salted = df_salted.repartition("chave_skew_salt")
```

---

## 4. Operações Perigosas: Overusing Collect

### O que é o Collect?

O método `.collect()` traz todos os dados do cluster para o driver. Isso pode causar:

- **Out of Memory**: Estouro de memória no driver.
- **Saturação de rede**: Transferência massiva de dados.
- **Falhas de aplicação**: Jobs travam ou falham.

### Boas Práticas

- Use `.show()`, `.take(n)` ou `.limit(n)` para visualizar amostras.
- Evite `.collect()` em grandes datasets.

#### Exemplo PySpark

```python
# Perigoso: pode travar o driver
dados = df.collect()

# Correto: visualizar apenas uma amostra
df.show(5)
amostra = df.limit(1000).toPandas()
```

---

## 5. Transformações Sequenciais: Loop Inadequado

### Problema

Usar loops para ler arquivos e apendar em DataFrames é ineficiente e vai contra o paradigma distribuído do Spark.

### Solução

- Use leitura em lote (`spark.read...`) para processar múltiplos arquivos de uma vez.
- Evite loops e apend manual.

#### Exemplo PySpark

```python
# Errado: loop para ler arquivos
dfs = []
for arquivo in lista_arquivos:
    dfs.append(spark.read.parquet(arquivo))
df_final = reduce(lambda a, b: a.union(b), dfs)

# Correto: leitura em lote
df_final = spark.read.parquet("/caminho/arquivos/*.parquet")
```

---

## 6. Joins no Spark: Estratégias e Performance

### Tipos de Join

- **Broadcast Hash Join (BHJ)**: Ideal para joins entre uma tabela pequena e uma grande. A tabela pequena é replicada em todos os executores.
- **Shuffle Hash Join (SHJ)**: Usado quando ambas as tabelas são grandes, mas uma é até 3x menor que a outra.
- **Shuffle Sort Merge Join (SSMJ)**: Usado quando ambas as tabelas são grandes e precisam ser ordenadas antes do join.

### Boas Práticas

- Prefira BHJ sempre que possível (tabelas pequenas).
- Ajuste o parâmetro `spark.sql.autoBroadcastJoinThreshold` para aumentar o limite do BHJ (ex: 1GB ou mais, se o cluster permitir).
- Use hints (`broadcast(df)`) para forçar BHJ quando necessário.
- Evite SSMJ, pois é o mais caro em termos de performance.

#### Exemplo PySpark

```python
from pyspark.sql.functions import broadcast

# Broadcast join explícito
df_join = df_grande.join(broadcast(df_pequeno), "chave")

# Ajustando o threshold para broadcast
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 1024*1024*1024)  # 1GB
```

---

## 7. Predicate Pushdown e Column Pruning

### O que são?

- **Predicate Pushdown**: O Spark filtra os dados o mais cedo possível, lendo apenas os arquivos/linhas necessárias.
- **Column Pruning**: O Spark lê apenas as colunas necessárias para a consulta.

### Benefícios

- Reduz o volume de dados lidos do storage.
- Melhora a performance das queries.
- Diminui o custo de I/O.

#### Exemplo PySpark

```python
# Predicate pushdown e column pruning automáticos ao selecionar colunas e filtrar
df = spark.read.parquet("/caminho/dados")
df_filtrado = df.filter("ano = 2023").select("id", "valor")
```

---

## 8. Coalesce vs Repartition

### Coalesce

- Reduz o número de partições.
- Minimiza o movimento de dados (menos shuffle).
- Ideal para otimizar escrita no storage (menos arquivos).

### Repartition

- Redistribui os dados entre as partições (gera shuffle).
- Pode aumentar ou diminuir o número de partições.
- Útil para balancear dados antes de operações pesadas.

### Quando usar cada um?

- **Coalesce**: Antes de escrever dados, para evitar small files.
- **Repartition**: Antes de operações que exigem balanceamento, como joins ou agregações.

#### Exemplo PySpark

```python
# Coalesce para reduzir arquivos na escrita
df.coalesce(10).write.parquet("/caminho/saida")

# Repartition para balancear dados antes de join
df = df.repartition(50, "chave")
```

---

## 9. Pandas API on Spark

- O Pandas API on Spark permite usar comandos pandas de forma distribuída.
- Ideal para quem já tem código em pandas e quer escalar para Big Data.
- No Spark 4.0, o suporte está ainda mais robusto.

#### Exemplo PySpark

```python
import pyspark.pandas as ps

psdf = ps.read_parquet("/caminho/dados")
psdf['nova_coluna'] = psdf['coluna'] * 2
psdf.groupby('outra_coluna').mean()
```

---

## 10. Considerações Finais

- **Não particione por padrão**: Só faça se realmente necessário.
- **Evite small files**: Use coalesce e monitore o tamanho dos arquivos.
- **Aproveite os recursos modernos**: Delta Lake, Iceberg, Liquid Clustering, Predicate Pushdown.
- **Monitore e ajuste**: Use ferramentas de monitoramento para identificar gargalos e ajustar parâmetros.

---

## Referências e Dicas

- **Documentação Spark**: https://spark.apache.org/docs/latest/
- **Delta Lake**: https://docs.delta.io/latest/
- **Iceberg**: https://iceberg.apache.org/
- **Ferramentas de monitoramento**: Spark UI, SparkMeasure

---

> **Resumo**: O Apache Spark evoluiu e, com ele, as melhores práticas para particionamento e performance. O que era regra no passado (particionar sempre) hoje deve ser analisado com cautela. Use as ferramentas e recursos modernos para garantir performance, escalabilidade e baixo custo.

