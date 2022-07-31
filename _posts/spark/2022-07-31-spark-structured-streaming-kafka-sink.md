---
layout: post
title:  "Apache Spark - Spark Structured Streaming Kafka Sink는 Exactly-Once를 지원하지 않는다"
date:   2022-07-31 14:00:00 +0900
author: leeyh0216
tags:
- apache-spark
- kafka
---

> Spark Structured Streaming Kafka Sink의 동작에 대해 알아봅니다. 기본적인 사용법보다는 내부적으로 Kafka Sink가 어떻게 동작하는지, 왜 Spark Structured Streaming Kafka Sink의 한계에 대한 이야기를 다루고 있습니다.

# Spark Structured Streaming Kafka Sink

회사에서 최근 Spark Structured Streaming을 사용하고 있고, 내가 개발하는 모듈은 Source와 Sink 모두 Kafka를 사용하고 있다. Lambda Architecture로 구성했기 때문에, 실시간 데이터의 경우는 어느정도 중복/유실이 발생할 수 있다고 생각하고 다음날 배치 작업에서 데이터를 Truncate하여 이전 일자의 데이터의 무결성을 보장하는 방향으로 진행하고 있다.

[Spark Structured Streaming Guide](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)의 Overview에는 아래와 같은 내용이 존재한다.

> the system ensures **end-to-end exactly-once fault-tolerance guarantees through checkpointing and Write-Ahead Logs**. In short, Structured Streaming provides fast, scalable, fault-tolerant, end-to-end exactly-once stream processing without the user having to reason about streaming

종단 간(end-to-end) 정확히 한번(Exactly-Once) 처리를 보장한다는 내용인데, 그럼 Kafka Source -> Kafka Sink의 과정에서도 동일하게 Exactly-Once가 보장되는 것일까?

[Spark Structured Streaming + Kafka Integration Guide](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)의 Writing Data to Kafka 섹션을 보면 아래와 같은 구문이 나와 있다.

> Take note that Apache Kafka only supports at least once write semantics.

Apache Kafka가 at-least-once Write Semantic 밖에 지원하지 않기 때문에, **Spark Structured Streaming 또한 Kafka Sink는 at-least-once를 보장**한다는 의미이다.

그런데 [KIP-98 - Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) 문서를 보면 Kafka에서는 Transaction 도입을 통해 exactly-once Semantic을 지원한다고 되어 있다.

과연 진실은 무엇일까?

## Spark Structured Streaming의 Kafka Sink

### Kafka Producer의 특징과 TX

내부 코드에 대해 분석하기 전에 Kafka Producer의 특징과 TX 적용 시의 유의사항에 대해 알아본다.

[KafkaProducer - JavaDoc](https://javadoc.io/doc/org.apache.kafka/kafka-clients/latest/index.html)을 보면 `KafkaProducer` 객체는 Thread-Safe하기 때문에, 여러 스레드에서 공유하여 사용해도 된다고 쓰여 있다.

만일 TX를 적용한 `KafkaProducer` 객체를 만들고 싶다면 어떨까? [Exactly Once Processing in Kafka in Java](https://www.baeldung.com/kafka-exactly-once) 문서를 참고하면 `KafkaProducer` 객체 단위로 TX를 관리하기 때문에, 트랜잭션 간에는 같은 `KafkaProducer` 객체를 공유하면 안된다고 되어 있다.

### Structured Streaming에서의 Kafka Sink 내부 구현

Structured Streaming에서 사용되는 Kafka 관련 Connector의 구현은 `org.apache.spark.sql.kafka010` 패키지 내에 존재한다. 이 중 주요하게 보아야 할 클래스는 다음과 같다.

* `InternalKafkaProducerPool`
* `KafkaDataWriter`
* `KafkaRowWriter`

클래스 별 역할과 주요 코드 확인 후 왜 이런 구성을 취했는지에 대한 질문/답변을 구성해본다.

#### `InternalKafkaProducerPool`

[`InternalKafkaProducerPool`](https://github.com/apache/spark/blob/master/connector/kafka-0-10-sql/src/main/scala/org/apache/spark/sql/kafka010/producer/InternalKafkaProducerPool.scala)은 `KafkaProducer`의 생성과 Pool 유지를 담당한다. 

```
private val cache = new mutable.HashMap[CacheKey, CachedProducerEntry]
```

`InternalKafkaProducerPool`은 내부적으로 1개 이상의 `KafkaProducer`를 Key 기준으로 유지한다. 여기서의 `CacheKey`는 커스텀 타입으로 지정되어 있으며, `private type CacheKey = Seq[(String, Object)]`이다. **캐시 키를 구성하는 `Seq[(String, Object)]`는 `KafkaProducer` 객체 초기화 시 설정하는 프로퍼티 목록**이다.

```
private[producer] def acquire(kafkaParams: ju.Map[String, Object]): CachedKafkaProducer = {
    val updatedKafkaProducerConfiguration =
      KafkaConfigUpdater("executor", kafkaParams.asScala.toMap)
        .setAuthenticationConfigIfNeeded()
        .build()
    val paramsSeq: Seq[(String, Object)] = paramsToSeq(updatedKafkaProducerConfiguration)
    synchronized {
      val entry = cache.getOrElseUpdate(paramsSeq, {
        val producer = createKafkaProducer(paramsSeq)
        val cachedProducer = new CachedKafkaProducer(paramsSeq, producer)
        new CachedProducerEntry(cachedProducer,
          TimeUnit.MILLISECONDS.toNanos(cacheExpireTimeoutMillis))
      })
      entry.handleBorrowed()
      entry.producer
    }
  }
```

클라이언트는 `KafkaProducer` 객체를 Pool로부터 가져오기 위해 `acquire`를 호출한다. 매개변수는 `KafkaProducer` 초기화 시 사용할 프로퍼티 목록이며 이는 위에서 말했듯 캐시 키로 사용된다. 여러 스레드가 동시에 `acquire`를 호출할 수 있기 때문에 `synchronized` 블록을 통해 동일 키에 대한 `KafkaProducer`가 1개만 생성될 수 있도록 처리한 것을 볼 수 있다.

```
private[kafka010] object InternalKafkaProducerPool extends Logging {
  private val pool = new InternalKafkaProducerPool(
    Option(SparkEnv.get).map(_.conf).getOrElse(new SparkConf()))
  ...
}
```

`InternalKafkaProducerPool`의 초기화에 대한 부분이다. Pool을 **싱글턴으로 유지**하기 위해 object 내에 `InternalKafkaProducerPool` 객체 하나를 초기화했고, `InternalKafkaProducerPool` 클래스의 생성자는 `private`인 것을 확인할 수 있다. 즉, JVM(Executor) 내에 무조건 한개의 `InternalKafkaProducerPool`만 존재한다. 결론적으로 **하나의 Executor에서 수행되는 동일 쿼리의 Task 들은 같은 `KafkaProducer`를 공유**한다.

위 코드 구성에 대해 내가 생각했던 질문과 답변은 아래와 같다.

##### JVM(Executor) 당 1개의 `KafkaProducer` 객체만 유지해도 되는 것 아닌가?

아니다. 하나의 `SparkSession` 내에서 1개 이상의 스트리밍 쿼리를 수행할 수 있고, 각각의 쿼리의 결과를 다른 Kafka Cluster, Topic, Configuration으로 쓸 수 있기 때문에, 쓰기 구성에 따라 1개 이상의 `KafkaProducer` 객체가 필요할 수 있다.

##### 그렇다면 쿼리들은 어떤 기준으로 `KafkaProducer`를 공유하는가?

DataFrameWriter의 `option` 구성이 같은 쿼리들은 같은 `KafkaProducer`를 공유한다. 그렇기 때문에 CacheKey를 `KafkaProducer` 생성에 필요한 프로퍼티 목록으로 구성한 것이다.

##### 코드를 보니 주기적으로 Eviction하는 과정이 있는데, 이는 왜 그런 것인가?

문서에도 나와 있지만 보안을 위해 Delegation Token이 주기적으로 갱신되고, `KafkaProducer`에 이 갱신된 Delegation Token을 적용해야 하기 때문에 주기적으로 Eviction 해주는 것이다.

이 또한 `InternalKafkaProducerPool`에서 수행하며, 내부에 `SingleThreadScheduledExecutor` 객체를 초기화하여 수행한다. 관련 설정은 `spark.kafka.producer.cache.timeout	`, `spark.kafka.producer.cache.evictorThreadRunInterval` 이다.

> `InternalKafkaProducerPool`의 설계 시의 가정이 Structured Streaming Kafka Sink에서 Exactly-Once를 보장하지 못하게 하는 하나의 원인이 되었다(물론 설계와 코드를 변경하면 되고, 이게 Exactly-Once를 보장하지 못하게 하는 핵심 원인은 아니다). 이 가정이 무엇인지는 다음 섹션에서 설명한다.

#### `KafkaDataWriter`

[`KafkaDataWriter`](https://github.com/apache/spark/blob/master/connector/kafka-0-10-sql/src/main/scala/org/apache/spark/sql/kafka010/KafkaDataWriter.scala)는 Spark Datasource V2 API를 구현한 클래스이며, 실질적으로 Row를 Sink하는 역할을 수행한다. **`KafkaDataWriter`는 파티션 별로 1개씩 생성**된다. 또한 `DataWriter`의 `write`, `commit`, `abort`, `close`를 구현한다.

```
def write(row: InternalRow): Unit = {
  checkForErrors()
  if (producer.isEmpty) {
    producer = Some(InternalKafkaProducerPool.acquire(producerParams))
  }
  producer.foreach { p => sendRow(row, p.producer) }
}
```

전달받은 데이터를 쓰는 `write` 메서드의 구현이다. `KafkaProducer`를 `InternalKafkaProducerPool`에서 얻어온 뒤, 이를 통해 데이터를 쓰는 `sendRow` 메서드를 호출하는 것을 확인할 수 있다. `sendRow` 메서드는 `KafkaDataWriter`가 상속받는 `KafkaRowWriter`에서 구현하고 있다. 이는 뒤에서 살펴본다.

> "왜 foreach로 호출하나요?"라는 질문을 할 수 있다. 마치 `N`개의 `producer`가 설정되어 있는 것처럼 보일 수 있는데, Scala에서 `Option`에 대한 `foreach`는 Option에 어떤 값이 설정(`Some`)되어 있다면 호출하고, 설정되어 있지 않다면(`None`) 호출되지 않는다는 의미다.

```
def commit(): WriterCommitMessage = {
  // Send is asynchronous, but we can't commit until all rows are actually in Kafka.
  // This requires flushing and then checking that no callbacks produced errors.
  // We also check for errors before to fail as soon as possible - the check is cheap.
  checkForErrors()
  producer.foreach(_.producer.flush())
  checkForErrors()
  KafkaDataWriterCommitMessage
}
```

Micro Batch의 Commit 시 호출되는 `commit` 메서드의 구현이다. 여기서 `KafkaProducer`의 `commit`을 호출하지 않고, `flush`만 호출한다. 즉, Structured Streaming Kafka Sink는 `commit`을 호출하지 않기 때문에, 각 Micro Batch의 Commit Phase에서 모든 데이터가 나갔다는 것(flush)만 보장한다.

#### `KafkaRowWriter`

이 클래스는 "KafkaWriteTask.scala" 파일 내에 구현되어 있으며, `KafkaDataWriter`가 상속받는다. 여기에 위에서 말한 `sendRow`의 실제 구현이 있다.

```
protected def sendRow(
  row: InternalRow, producer: KafkaProducer[Array[Byte], Array[Byte]]): Unit = {
    val projectedRow = projection(row)
    ...
    val record = if (projectedRow.isNullAt(3)) {
      ...
    producer.send(record, callback)
  }
```

이 메서드는 `InternalRow`를 적절히 변형하여 `KafkaProducer`의 `send` 메서드를 통해 전송한다. 여기서 주의해야 할 점은 `KafkaProducer`의 `send` 메서드는 비동기(asynchronous)하는 것이다. 그렇기 때문에 `KafkaDataWriter`의 `commit`에서 실제 `KafkaProducer`의 `commit`은 호출하지 않더라도 지금까지 쓴 데이터를 모두 Kafka Broker에 전송했다는 것만은 보장하기 위해 `flush`를 호출하는 것이다.

### 왜 Structured Streaming Kafka Sink는 Exactly-Once를 지원할 수 없는 구조인가?

두 가지 이유가 존재한다.

* Spark에서 Two-Phase Commit을 지원하지 않는다.
* Kafka TX Commit 단위와 Spark의 `KafkaProducer` 공유 단위의 불일치

좀 더 간단한 두 번째 이유를 설명한 뒤, 첫 번째 이유를 설명한다.

#### KafkaProducer TX 단위와 Spark의 `KafkaProducer` 공유 단위의 불일치

`KafkaDataWriter`는 파티션 별로 한개씩 생성되며 `KafkaDataWriter` 별로 `commit` 메서드를 수행한다. 즉, 하나의 Micro Batch 쓰기 과정에서 파티션 개수만큼의 `commit`이 발생하는 것이다. 결과적으로 `KafkaProducer`는 `KafkaDataWriter` 단위로 할당되어야 한다.

그러나 위에서 말했듯 `InternalKafkaProducerPool`은 동일한 프로퍼티를 기준으로 `KafkaProducer`를 공유한다. 즉 하나의 Executor 내에 같은 쿼리를 수행하는 Task(파티션)는 동일한 `KafkaProducer`를 사용하게 된다.

만일 `InternalKafkaProducerPool`이 `KafkaProducer`를 파티션 별로 하나씩 만들어준다 해도, Spark의 쓰기 옵션 구성때문에 정상적으로 동작하지 않을 것이다.

```
df
  .selectExpr("topic", "CAST(key AS STRING)", "CAST(value AS STRING)")
  .writeStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2")
  .option("kafka.transactional.id", UUID.randomUUID().toString())
  .start()
```

위와 같이 `kafka.transactional.id` 값으로 `UUID.randomUUID().toString()`을 전달했다 해도, 이는 Driver에서 수행되는 코드이기 때문에 전체 Executor에게 같은 UUID가 전달되고, 결과적으로 모든 `KafkaProducer`들이 같은 TX ID를 공유하기 때문에 정상적인 코드 진행이 불가하다.

이를 해결하기 위해서는 TX ID를 바로 전달하는 것이 아니라 prefix 정도만 전달하고, `InternalKafkaProducerPool` 등에서 이 Prefix에 Partition ID 등을 붙여주는 방식으로 해결되어야 할 것이다.

다만 이 문제는 근본적인 원인에 비해 상대적으로 해결이 쉽다.

#### Spark에서 Two-Phase Commit을 지원하지 않는다.

사실 이게 근본적인 원인이다. Spark에서 Two-Phase Commit을 지원하지 않는다는 것이다.

Spark에서 Custom Connector를 구현하기 위해서는 [DataWriter](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/java/org/apache/spark/sql/connector/write/DataWriter.java) 인터페이스를 구현해야 하고, 여기에는 `commit`과 `abort`가 있다.

여기서 발생하는 문제는 Spark은 "분산" 프레임워크이기 때문에, 트랜잭션의 커밋 또한 "분산"으로 실행되어야 한다는 것이다. 예를 들어 아래와 같은 상황을 생각해보자.

* 3개의 쓰기 파티션으로 구성된 작업이 존재
* `DataWriter`의 `commit` 호출 시 파티션 별로 `KafkaProducer`의 `commit` 수행

그런데 특정 시점에 2개의 쓰기 파티션은 성공했는데, 1개의 쓰기 파티션에서는 오류가 발생했다면? 이미 2개 파티션의 데이터가 Kafka Broker에 완전히 반영되었기 때문에 Rollback이 불가능하다.

이처럼 분산 환경에서는 1개의 트랜잭션이 N개의 노드에 나뉘어 실행되기 때문에 일반적인 커밋이 아니라 Two-Phase Commit 같은 프로토콜을 사용해야 한다. 

결론은 Spark은 애초에 Two-Phase Commit을 지원하지 않기 때문에 Connector의 구현에서도 commit을 사용하지 않고 flush만 시키는 것이다. 결과적으로 일부 노드의 실패가 발생하는 경우 중복이 발생 가능한 Exactly-Once Semantic을 지원하는 것이다.

## 정리

[Spark-29808 Implement Kafka EOS sink for Structured Streaming](https://github.com/apache/spark/pull/25618)에서 Spark Structured Streaming에 Kafka TX를 적용하려 했던 시도가 있었다. 스레드를 읽어보면 여기서도 Spark에서 2PC를 지원하지 않고, Simple 2PC를 구현해서 처리하려는 노력을 하고 있지만, 결과적으로 진행되지 않고 티켓이 종료된 것을 볼 수 있다.

[Apache Kafka transactional writer with foreach sink, is it possible?](https://www.waitingforcode.com/apache-spark-structured-streaming/apache-kafka-transactional-writer-foreach-sink-is-posible/read) 이라는 글에서 단순히 Kafka TX Commit을 수행하는 로직을 구현했지만, 이 또한 2PC의 구현이 아니기 때문에 중복 문제는 피할 수 없다.

만일 Kafka Sink에서 반드시 Exactly Once를 지원해야 한다면 [Apache Flink와 같이 2PC](https://flink.apache.org/features/2018/03/01/end-to-end-exactly-once-apache-flink.html)를 지원하는 프레임워크를 사용하는 방법 밖에 없다.

물론 Spark에서도 동일 Key에 대한 State를 유지하여 중복 제거를 하라곤 하지만... 잘 될까..?