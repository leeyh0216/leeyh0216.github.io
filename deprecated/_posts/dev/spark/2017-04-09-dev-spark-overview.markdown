---
layout: post
title:  "[Spark] Spark Overview"
date:   2017-04-09 13:14:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Spark 개요를 정리하기 위해 작성한 문서입니다.

### Hadoop MapReduce
2009년 하둡의 0.20.1 버전이 공개되고, HDFS와 MapReduce를 이용한 분산 처리 프로그램을 개발할 수 있게 되었다. 기존의 데이터 처리에서는 프로그램이 있는 곳으로 데이터가 이동하였지만, HDFS의 Locality를 이용하여 데이터가 있는 곳으로 프로그램이 이동하게 되었으며, 이로 인해 빠른 처리속도를 얻을 수 있게 되었다.

하지만 이러한 MapReduce 프레임워크에도 한계가 있었다. MapReduce의 경우 파일 시스템을 기반으로 대부분의 동작이 이루어졌기 때문에, 인메모리 기반 프레임워크보다 속도가 느릴 수 밖에 없었고, Map과 Reduce 연산만으로 프로그래밍을 진행해야 했기 때문에, 프로그램 구조에 한계가 있을 수 밖에 없었다.

### Spark
위와 같은 하둡의 단점을 보완하기 위해서 UC Berkeley 대학에서 연구로 시작되어 2012년 Spark의 핵심 개념인 RDD에 대한 논문이 발표되었다. Spark의 경우 2개의 함수(Map과 Reduce) 밖에 지원하지 않는 하둡 MapReduce보다 자연스럽고 강력한 데이터 처리 함수를 제공하여 프로그램의 복잡도를 낮추었고, Spark Streaming, SparkSQL, GraphX, SparkR, Mlib 등을 제공하여 분야를 넓혀 왔다. 

#### RDD 개요
일반적으로 국내 서적에서는 RDD를 다음과 같이 정의한다.

1. Spark가 사용하는 핵심 데이터 모델로써, 다수의 서버에 걸쳐 분산 방식으로 저장된 데이터 요소들의 집합.
2. 각 RDD는 병렬 처리가 가능하다.
3. RDD는 Partition 단위로 나뉘어져있으며, 작업을 수행할 때 Partition 단위로 나누어 작업하며, 각 Parition들은 네트워크를 통해 다른 서버로 이동할 수 있다. 이를 Shuffle이라고 한다.
4. RDD는 내부적으로 Lineage를 가지고 있어, 자신이 어떠한 방식으로 생성되었는지 기록하고, 만일 오류가 발생하여 RDD 내의 데이터가 유실되어도 Lineage를 통하여 동일한 RDD를 생성할 수 있다.

하지만 Spark Contributer인 [A.Grishchenko가 작성한 SlideShare](https://www.slideshare.net/AGrishchenko/apache-spark-architecture?ref=https://0x0fff.com/category/spark/)를 읽어보면 다음과 같이 자세한 정보를 얻을 수 있다.

1. RDD는 Data Transformation의 Interface이다.
2. RDD는 유지되는 저장소(HDFS, Cassandra, HBase , etc..), Cache(memory, memory+disk ...) 또는 다른 RDD를 의미한다.
3. 각 Partition들은 오류 또는 메모리에서 제거되었더라도 다시 생성될 수 있다.
4. Metadata를 저장하기 위한 Interface이며 다음과 같은 내용을 저장한다.
 - Partitions : 해당 RDD에 연관된 Data의 조각
 - Dependencies : 자신의 생성에 연관된 부모 RDD의 리스트
 - Compute : 부모 RDD가 자신을 생성할 때 사용한 Function
 - Preferred Location : Compute 수행을 위해 가장 적합한 위치(Data Locality)
 - Partitioner : Data를 어떻게 Partition으로 나누는지에 대한 정보

RDD는 Spark에서 데이터를 생성하기 위한 유일한 도구이며, Transformation과 Action 두가지로 이루어져 있다.
또한 Lazy Evaluation을 채용하여 반드시 필요한 연산만을 수행하게 한다.

#### DAG
DAG(Directed Acyclic Graph)의 줄임말로써, Graph의 한 종류이다. Graph와 동일하게 Edge, Vertex 로 이루어져 있으며, Edge의 경우 방향성이 있고, Graph 내에 Cycle이 생성될 수 없다.
이러한 개념을 이용하여 Spark에서는 DAG Scheduler라는 컴포넌트를 만들었다. DAG Scheduler는 Spark의 작업(Transformation, Action)을 어떻게 수행해야 하는지를 정의한 지도라고 표현할 수 있다.
DAG의 각 Component는 Spark와 다음과 같은 관계를 맺고 있다.
- Node : RDD Parition
- Edge : Transformation
- Acyclic : A -> B -> A와 같이 생성 이전의(부모) RDD로 돌아갈 수 없는 제약조건
- Direct : Transformation은 State를 변경하는 작업이라는 의미

Tensorflow에서도 동일한 개념을 사용하여 작업을 구성하는 걸 보면 DAG는 작업 스케쥴링에 적합한 모델이라고 볼 수 있다.

이러한 개념과 더불어 Stage와 Task라는 개념도 존재한다.
- Stage : 하나의 독립 Worker에서 연결되어 실행될 수 있는 Transformation의 집합. 각 Stage는 Action이나 Shuffle 작업으로 끝나게 된다.
- Task : 각 Stage를 이루는 작업. 스케쥴링의 기본 단위이며 병렬성을 가지고 있다.

일반적으로 생각할 때 Task의 단위가 map, flatMap 등의 Transformation이라고 생각하지만, Task로 나뉘는 기준은 Partition이다. 다음과 같은 예를 볼 때,
{% highlight scala %}
val a : RDD[String] = sc.parallize(List("a","b","c"))
a.map( some function ).flatMap( some function ).reduceByKey( some function ).foreach( some function )
{% endhighlight %}
일반적으로 map Task, flatMap Task, reduceByKey Task 로 나뉘어진다고 생각하지만 그렇지 않다.
만일 a RDD가 3개의 Partition(a,b,c)으로 나뉘어진다고 가정할 때 "a" Partition이 수행하는 작업 Pipe인 map,flatMap, reduceByKey가 Task1, "b" Parition이 수행하는 작업 Pipe인 map,flatMap, reduceByKey가 Task2 가 되는 형식이다. 다른 Partition과 관계가 없는 작업이므로 병렬성을 보장받을 수 있으므로, 하나의 Stage를 이루는 Task는 위와 같은 형식으로 병렬로 수행된다.

여기서 나오는 개념이 하나 더 있는데, Narrow Dependencies와 Wide Dependencies이다. Narrow Dependencies의 경우, Source Partition과 Dest Partition이 1:1 관계를 가지는 map, flatMap 등의 연산이다. 이는 다른 Partition과 관계 없이 병렬적으로 진행 될 수 있기 때문에 Task로 취급한다.
하지만 Wide Dependencies의 경우 Source Partition과 Dest Partition이 N:1 관계를 가지는 reduceByKey, groupByKey와 같은 연산이다. 이는 다른 Partition을 조합하여 생성해야 하기 때문에 병렬성을 보장받을 수 없고, Stage의 기준이 된다.
Wide Dependencies의 경우 모든 서버에 퍼져있는 Partition 들을 조합하여야 하기 때문에 Shuffle이 발생하게 되고, 이는 Network를 통해 이루어지므로 속도가 떨어지게 되어있다.
즉, 최대한 Wide Dependencies를 줄이는 것이 속도를 올리는 방법이라고 생각한다.

#### Spark Application 구성요소
Spark Application은 다음과 같은 구성요소로 이루어져 있다.
- Driver : Spark Application의 Entry Point이며, RDD를 DAG로 변경하고, DAG를 다시 Stage로 변경하여 Worker Node의 Executor에게 해당 Stage의 실행을 요청하고, 이 모든 과정에서 발생하는 Metadata를 저장하고 있다.
- Executor : 실제로 Data Processing을 수행하고, 데이터를 외부 소스에서 읽고 기록한다. 데이터는 Heap(2.x 버전에서는 변경됨 -> Tungsten Project)과 HDD에 저장된다.
