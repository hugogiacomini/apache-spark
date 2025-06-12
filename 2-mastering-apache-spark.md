
# Chapter 01) Advanced Pillars

## Arquitetura do Spark

O Apache Spark é uma plataforma de computação distribuída que utiliza um modelo de execução baseado em clusters. Sua arquitetura é composta pelos seguintes componentes principais:

- **Driver e Executores**:  
    O driver é responsável por receber a aplicação, criar um plano de execução lógico e físico, e gerar tarefas e estágios que serão executados pelos executores.  
    Os executores são responsáveis por executar as tarefas atribuídas pelo driver e retornar os resultados.

- **Cluster Manager**:  
    O cluster manager gerencia os recursos de execução do cluster. Existem diferentes tipos de cluster managers suportados pelo Spark, como Yarn, Kubernetes, Mesos e Spark Standalone.

## Alocação de Recursos

A alocação de recursos no Spark é baseada em partições, que são a unidade de paralelismo. Algumas recomendações importantes incluem:

- Utilizar de 3 a 4 vezes o número de cores disponíveis em partições para maximizar o paralelismo.
- Threads e slots controlam a execução das tarefas nos executores.

## Internals do Spark

### Execução do Spark

O processo de execução no Spark segue as etapas abaixo:

1. **Spark Driver**:  
     Recebe a aplicação, realiza o parse, binding e cria um plano lógico.

2. **Catalyst Optimizer**:  
     Analisa o plano lógico e gera um plano otimizado.

3. **Physical Planner**:  
     Gera o plano físico de execução, que é dividido em estágios e tarefas.

4. **Task Scheduler**:  
     Agenda a execução das tarefas nos executores.

### RDD (Resilient Distributed Dataset)

O RDD é o modelo de programação de baixo nível do Spark. Ele é uma estrutura de dados distribuída e imutável, mas foi substituído pelas APIs de alto nível, como DataFrame e Dataset.

### DataFrame e Dataset

As APIs de alto nível do Spark oferecem suporte a estruturas de dados tabulares e utilizam o Catalyst Optimizer para gerar planos de execução otimizados. O Dataset, além disso, permite a definição de tipos customizados.

## Otimizações do Spark

- **Whole Stage Code Generation**:  
    Parte do Tungsten Project, essa otimização gera um único bytecode para executar múltiplas transformações.

- **Adaptive Query Execution (AQE)**:  
    Realiza otimizações em tempo de execução, como reparticionamento, otimização de joins e tratamento de stragglers.

## Questões relevantes

### Por que PySpark é mais rápido que RDD?

O PySpark utiliza APIs de alto nível, como DataFrame e Dataset, que são otimizadas pelo Catalyst Optimizer. Essas APIs permitem que o Spark gere planos de execução mais eficientes, aproveitando otimizações como o Whole Stage Code Generation e o Adaptive Query Execution (AQE). Por outro lado, o RDD é uma API de baixo nível que não se beneficia dessas otimizações, resultando em uma execução menos eficiente. Além disso, o PySpark reduz a sobrecarga de gerenciamento manual de partições e tarefas, simplificando o desenvolvimento e melhorando o desempenho.

### Como as JVMs são divididas no Spark?

As JVMs no Spark são divididas entre o Driver e os Executores. O Driver é responsável por coordenar a execução da aplicação, gerenciando o plano de execução e distribuindo tarefas para os Executores. Cada Executor, por sua vez, é uma JVM separada que executa as tarefas atribuídas pelo Driver e armazena os dados em memória ou disco local. Essa separação permite que o Spark escale horizontalmente, distribuindo a carga de trabalho entre várias JVMs em um cluster.

### O que são slots?

Slots são as unidades de execução disponíveis em cada executor no cluster do Spark. Cada slot pode executar uma tarefa por vez, e o número total de slots disponíveis em um cluster é determinado pelo número de núcleos (cores) alocados para os executores. A configuração adequada do número de slots é crucial para garantir o uso eficiente dos recursos do cluster e maximizar o paralelismo na execução das tarefas.

## Conclusão

Compreender os fundamentos internos do Spark é essencial para dominar a ferramenta. Este treinamento aprofundará conceitos como particionamento, alocação de recursos, planos de execução e otimizações do Spark.
