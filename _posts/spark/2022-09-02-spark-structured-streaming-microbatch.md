---
layout: post
title:  "Spark Structured Streaming의 MicroBatch 동작 원리 알아보기"
date:   2022-09-02 23:55:00 +0900
author: leeyh0216
tags:
- apache-spark
- kafka
---

> 이 글의 내용은 [MicroBatchExecution - Stream Execution Engine of Micro-Batch Stream Processing](https://jaceklaskowski.gitbooks.io/spark-structured-streaming/content/spark-sql-streaming-MicroBatchExecution.html)을 참고하여 작성하였습니다.

# Spark Structured Streaming의 MicroBatch 동작 원리 알아보기

MicroBatch 기반의 Spark Structured Streaming이 어떻게 동작하는지 알아본다.

## MicroBatch Stream Processing

[Wikipedia - Streaming Data](https://en.wikipedia.org/wiki/Streaming_data)에서 정의한 스트리밍 데이터는 "서로 다른 소스에서 발생하는 한정되지 않고(Unbounded), 연속적인(Continuous) 데이터"이다.

이러한 스트리밍 데이터를 처리하는 방식은 크게 Native 방식과 Micro Batch 방식으로 나뉜다.
* Native: 유입되는 레코드를 도착하는 시점에 하나씩 처리한다.
* Micro Batch: 유입되는 레코드를 특정 주기로 모아서 한번에 처리한다.

Micro Batch 방식은 레코드를 한번에 모아서 처리하기 때문에 상대적으로 Native에 비해 Latency가 크다는 단점을 가진다.

데이터 처리 프레임워크 중 Apache Flink, Storm가 Native Processing 방식을 지원하고, Apache Spark의 경우 Native(Continuous Streaming), Micro Batch 방식 모두를 지원하고 있다.

> 다만 Apache Spark의 Native 방식의 경우 Expermiental이며, Static Source와의 Join이 불가능하다는 제약사항 등이 존재한다.

이 글에서는 간단히 아래와 같은 코드를 실행했다고 가정하고, 실행 순서를 확인해본다.

```
df.writeStream
  .format("console")
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .start()
```

## Spark Structured Streaming에서의 Micro Batch의 구현

```
abstract class StreamExecution(
    override val sparkSession: SparkSession,
    override val name: String,
    val resolvedCheckpointRoot: String,
    val analyzedPlan: LogicalPlan,
    val sink: Table,
    val trigger: Trigger,
    val triggerClock: Clock,
    val outputMode: OutputMode,
    deleteCheckpointOnStop: Boolean)
  extends StreamingQuery with ProgressReporter with Logging
```

Spark Structured Streaming은 Streaming Query의 실행을 [`StreamExecution`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/StreamExecution.scala)이라는 클래스로 추상화했다. 생성자 시그니처에서 확인할 수 있듯 Streaming Query 실행에 필요한 기본적인 정보(Query Plan, Sink 대상 정보, Trigger 정보 등)를 포함하고 있다.

`StreamExecution`은 [`ContinuousExecution`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/continuous/ContinuousExecution.scala)과 [`MicroBatchExecution`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/MicroBatchExecution.scala)이 구현하고 있으며, 상속 구조와 세부 내용은 아래와 같다.

* `StreamExecution`: Streaming Query 실행 정보/로직을 추상화한 클래스
  * `MicroBatchExecution`: Micro Batch 방식의 Streaming Query 실행에 사용되는 클래스. `Trigger.processingTime`으로 쿼리의 Trigger를 초기화했을 때 선택된다.
  * `ContinuousExecution`: Continuous 방식의 Streaming Query 실행에 사용되는 클래스. `Trigger.Continuous`으로 쿼리의 Trigger를 초기화했을 때 선택된다.

> ```
> df.writeStream
>   .format("console")
>   .trigger(Trigger.ProcessingTime("2 seconds")) //이 부분에서 StreamExecution의 종류가 결정
>   .start()
> ```
> 
> 위 코드의 주석에도 쓰여 있듯, `trigger` 메서드가 `StreamExecution`의 종류를 결정한다.

`StreamExecution`의 종류가 결정된 이후 `start` 메서드를 호출하면, 내부적으로 `StreamExecution`의 `runActivatedStream`이 호출되며 실질적인 처리가 시작된다.

### MicroBatch의 주기적 실행을 담당하는 `TriggerExecutor`

`runActivatedStream`의 내부를 알아보기 전 MicroBatch의 주기적인 실행을 담당하는 `TriggerExecutor`에 대해 알아보기로 한다.

`MicroBatchExecution`에서는 생성자로 전달받은 `Trigger` 종류에 따라 쿼리를 주기적으로(혹은 한번만) 실행하는 `TriggerExecutor`를 초기화한다.

```
private val triggerExecutor = trigger match {
  case t: ProcessingTimeTrigger => ProcessingTimeExecutor(t, triggerClock)
  case OneTimeTrigger => OneTimeExecutor()
  case _ => throw new IllegalStateException(s"Unknown type of trigger: $trigger")
}
```

[`TriggerExecutor`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/TriggerExecutor.scala)는 매개변수로 전달받은 `Unit` => `Boolean` 타입의 함수를 주기적으로(혹은 한번만) 실행하는 `execute`라는 메서드를 선언하고 있으며, 이를 상속하는 `ProcessingTimeTrigger`와 `OneTimeTrigger`는 각각 이를 구현하고 있다.

```
trait TriggerExecutor {
  /**
   * Execute batches using `batchRunner`. If `batchRunner` runs `false`, terminate the execution.
   */
  def execute(batchRunner: () => Boolean): Unit
}
```

전달받은 함수의 결과 타입은 `Boolean`이며, 주석에도 나와 있듯 이 값이 `false`로 전달되었을 때는 호출을 중지해야 한다.(사용자가 쿼리를 terminate 했거나, 마지막 Offset까지 가져온 경우에 `false`가 반환된다)

> "Streaming Query인데 마지막 Offset이 어디있어?" 라는 질문을 할 수도 있다.
>
> 그러나 [Spark Structured Streaming + Kafka Integration](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html) 문서의 `endingOffsets`를 보면 마지막 Offset을 지정할 수 있는 것을 확인할 수 있다.

```
case class OneTimeExecutor() extends TriggerExecutor {
  override def execute(batchRunner: () => Boolean): Unit = batchRunner()
}
```

[`OneTimeExecutor`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/TriggerExecutor.scala#L34)는 말 그대로 "딱 한번만" 실행하는 방식으로써, 전달받은 함수를 한번 호출하고 종료한다.

```
case class ProcessingTimeExecutor(
    processingTimeTrigger: ProcessingTimeTrigger,
    clock: Clock = new SystemClock())
  extends TriggerExecutor with Logging {

  private val intervalMs = processingTimeTrigger.intervalMs
  require(intervalMs >= 0)

  override def execute(triggerHandler: () => Boolean): Unit = {
    while (true) {
      val triggerTimeMs = clock.getTimeMillis
      val nextTriggerTimeMs = nextBatchTime(triggerTimeMs)
      val terminated = !triggerHandler()
      ...
    }
  }
}
```

[`ProcessingTimeExecutor`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/TriggerExecutor.scala#L45)는 `Trigger` 설정을 통해 전달받은 주기 별로 함수를 실행한다. 내부적으로는 위와 같이 무한 루프를 돌며 전달받은 함수를 실행하는 것을 볼 수 있다.

### Streaming Query 실행을 담당하는 `runActivatedStream`

```
df.writeStream
  .format("console")
  .trigger(Trigger.ProcessingTime("2 seconds"))
  .start() //이 부분
```

위 코드의 `start` 메서드를 호출하면 생성된 `MicroBatchExecution`의 [`runActivatedStream`](https://github.com/apache/spark/blob/v3.2.2/sql/core/src/main/scala/org/apache/spark/sql/execution/streaming/MicroBatchExecution.scala#L182) 메서드가 호출된다. 

이 메서드에서는 실행 후 `triggerExecutor`의 `execute` 메서드에 Streaming Query 실행 로직을 전달하므로써 주기적으로 Micro Batch Query를 수행한다.

```
protected def runActivatedStream(sparkSessionForStream: SparkSession): Unit = {

    val noDataBatchesEnabled =
      sparkSessionForStream.sessionState.conf.streamingNoDataMicroBatchesEnabled

    triggerExecutor.execute(() => {
      ...//Streaming Query 실행 로직
    }
}
```

위의 `TriggerExecutor`에서 확인한 것처럼 `execute`에 전달된 함수는 Streaming Application이 중지되거나, 사용자가 쿼리를 중지하기 전까지 반복적으로 실행되는 로직이다. 내부 로직이 길기 때문에 큰 기능 단위로 나누어 분석해 보았다.

#### Micro Batch Loop의 첫 번째 실행과 초기 정보 설정

Micro Batch를 실행하는 Loop의 첫 번째 실행은 이후의 다른 실행들과 약간 다른 처리가 존재한다. 기존 실행 이력(Checkpoint에 기록된)을 기반(없다면 사용자의 설정)으로 초기 정보(Starting Offset 등)를 설정해야 하기 때문이다.

`StreamExecution`에는 `currentBatchId`라는 Long 타입 변수가 존재하는데, 이 값은 각 배치에 할당되는 ID로써 0-based로 1씩 단조 증가하는 값이다. -1로 초기화한 상태에서 실행되므로 이 값이 -1이라면 Loop의 첫번째 실행이라는 것을 판단할 수 있다.

```
if (currentBatchId < 0) {
  populateStartOffsets(sparkSessionForStream)
  logInfo(s"Stream started from $committedOffsets")
}
```

위와 같이 `currentBatchId`가 0 미만의 값인 경우에만 `populateStartOffsets`라는 메서드를 실행한다. 매 Micro Batch 마다 `currentBatchId`가 계속 1씩 증가므로 최초 실행 시에만 `populateStartOffsets`가 실행된다.

`populateStartOffsets` 메서드는 Checkpoint에 기록된 Offset Log(WAL), Commit Log를 기반으로
* 실행할 배치 ID(`currentBatchId`)
* 마지막으로 커밋된 Offset 정보(`committedOffsets`)
* 이번 배치에서 처리해야할 Offset 정보(`availableOffsets`)
를 초기화한다.

> 일부 케이스에서는 `committedOffsets`나 `availableOffsets`를 초기화할 수 없는 경우도 존재한다. 이러한 경우 내부 변수인 `isCurrentBatchConstructured`를 `false`로 설정한 뒤, 다음 실행되는 `constructNextBatch` 메서드(`isCurrentBatchConstructured`가 `false`인 경우에만 실행)에서 초기화할 수 있도록 처리된다.

메서드 내부에 분기 처리가 많은데, 대략적인 분기와 해당 분기에서 설정/처리되는 로직, 정보는 아래와 같다.

> 위에서 말했듯 배치에는 Long 타입의 ID가 부여되며, 매 배치가 끝날 때마다 배치 ID와 동일한 ID를 가지는 Commit이 생성된다는 것을 알고 아래 내용을 읽어본다!

##### Case 1. Offset Log에 마지막 실행 정보가 존재하는 경우

Checkpoint Location에 이전 애플리케이션에서 실행하던 Streaming Query에 대한 정보가 남아 있는 상황이다. Checkpoint의 Offset Log를 기반으로 아래 정보들을 초기화 한 뒤, Commit Log를 기반으로 아래 정보들을 보정(업데이트)한다.

> 위에서 간략히 설명했지만 `committedOffsets`와 `availableOffsets`라는 용어가 등장한다.
>
> 한 배치의 `committedOffsets`와 `availableOffsets`는 같다. 다만 `committedOffsets`는 배치가 성공적으로 수행된 후 기록되며 `availableOffsets`는 배치 수행 전에 미리 기록된다(Write Ahead Log).
>
> 결과적으로 N번째 Micro Batch에서 수행해야 하는 Offset의 범위는 N-1 번째 Micro Batch의 `committedOffsets` ~ N 번째 Micro Batch의 `availableOffsets` 까지이다.

* `currentBatchId`: Offset Log에 남아 있는 마지막 Batch ID로 설정
* `isCurrentBatchConstructured`: 우선 `true`로 설정.(다음 배치 정보를 초기화할 필요 없다는 의미)
* `availableOffsets`: 실행되었던 마지막 Batch의 `availableOffsets`로 설정
* `committedOffsets`: 실행되었던 마지막 Batch ID가 0이 아닌 경우(즉, 마지막 전에도 1개 이상의 배치가 존재했던 경우), Last Batch ID - 1 번째의 Batch의 `availableOffsets`로 설정된다.

위와 같이 Offset Log로 마지막 배치 실행 시의 상황을 대략적으로 추정한 뒤, Commit Log를 기반으로 아래와 같이 보정을 진행한다. 

* Offset Log에 기록된 마지막 배치 ID와 Commit Log에 기록된 커밋 ID가 일치하는 경우: 기존 애플리케이션의 마지막 배치가 정상 커밋되어 Gracefully Shutdown 된 경우이다.
  * 새로운 배치를 준비(`constructNextBatch`의 수행)할 수 있도록 `isCurrentBatchConstructed`를 `false`로 설정한다.
  * 마지막 배치가 정상 수행된 것이므로 `committedOffsets`에 마지막 배치의 `availableOffsets`를 추가한다.
* Offset Log에 기록된 마지막 배치 ID가 Commit Log에 기록된 커밋 ID보다 1이 큰 경우: 마지막 배치가 실행 중 비정상 Shutdown 된 경우이다.
  * 이 경우 새로운 배치를 준비할 필요가 없다(즉, `isCurrentBatchConstructured`를 그대로 `true`로 두어도 된다). 마지막 배치가 처리하려던 `availableOffsets`까지 처리를 수행하면 되기 때문이다.
  * `availableOffsets`와 `committedOffsets`도 기존과 동일하게 두면 된다.
* 그 이외의 경우: 별도 처리를 하지 않는다.

##### Case 2. Offset Log에 마지막 실행 정보가 존재하지 않는 경우

기존에 실행된 적 없는 Streaming Query의 실행이다. `currentBatchId`가 0으로 설정되며, `isCurrentBatchConstructured`가 초기 값인 `false`로 남아 있게 된다.

#### 다음 배치 실행 여부를 결정하는 `constructNextBatch` 메서드

매 Loop에서는 처리해야 할 새로운 데이터의 존재 여부를 확인하여 존재하는 경우에만 새로운 Micro Batch를 생성한다. 이 때 새로운 데이터의 존재 여부를 확인하는 메서드가 `constructNextBatch` 메서드이다.

```
if (!isCurrentBatchConstructed) {
  isCurrentBatchConstructed = constructNextBatch(noDataBatchesEnabled)
}
```

`constructNextBatch` 메서드는 위와 같이 `isCurrentBatchConstructured`가 `false`일 때만 실행되는데, 위에서도 한번 설명했지만 `isCurrentBatchConstructured`가 어떨 때 `true`이고, 어떨 때 `false`로 설정되는지 다시 한번 정리해본다.

* `true`로 설정되는 경우: 처리할 데이터가 있는 것을 이미 인지한 경우
  * 최초 Loop 실행에서 완료되지 않은 이전 배치의 기록이 존재하는 경우
* `false`로 설정되는 경우
  * 최초 Loop 실행에서 이전 배치가 정상적으로 완료된 경우
  * 최초 Loop 실행에서 이전에 Streaming Query가 수행된 적이 없는 경우(Checkpoint Location이 깔끔한 경우)

`constructNextBatch` 내에서는 Streaming Query에서 사용되는 Source들의 현 시점 마지막 Offset을 가져와서 기존 Offset들과 비교하여 새로운 데이터 존재 여부를 확인하는 메서드(`isNewDataAvailable`)의 실행 결과와 `lastExecutionRequiresAnotherBatch` 값을 결합(OR 연산)하여 다음 배치를 실행해야하는지 여부(`shouldConstructNextBatch`)를 결정한다.

> 주의해야 할 점은 처리할 새로운 데이터가 존재하지 않더라도(`isNewDataAvailable`이 `false`로 설정되어 있더라도), `lastExecutionRequiredAnotherBatch` 값이 `true`인 경우 Micro Batch 자체는 생성된다는 점이다

`shouldConstructNextBatch`는 `constructNextBatch`의 반환 값이며, 반환 전에 해당 값이 `true`인 경우 WAL에 이번 배치에 처리해야하는 Offset 정보(위에서 소스 별로 가져온 현 시점 마지막 Offset 정보들)를 기록한다.

#### 1 단위 Micro Batch의 실행 - `runBatch`

```
currentBatchHasNewData = isNewDataAvailable

currentStatus = currentStatus.copy(isDataAvailable = isNewDataAvailable)
if (isCurrentBatchConstructed) {
  if (currentBatchHasNewData) updateStatusMessage("Processing new data")
  else updateStatusMessage("No new data but cleaning up state")
  runBatch(sparkSessionForStream)
} else {
  updateStatusMessage("Waiting for data to arrive")
}
```

`isCurrentBatchConstructured`이 `true`인 경우 1개 단위의 Micro Batch를 만들고 이를 실행하는 `runBatch` 메서드가 실행된다. `runBatch`의 경우 스트리밍 소스 데이터와 사용자 쿼리를 기반으로 Plan을 생성하고, 이를 Sink에 기록하는 일련의 과정을 포함하고 있는데, 대략적으로 아래와 같다.

1. 모든 스트리밍 소스에 대해 처리할 데이터 범위(Offset)를 기반으로 데이터 프레임을 생성한다.
  * 이 부분이 `SQL` 탭에서 `getBatch`라고 기록되는 Phase이다.
  * 처리할 오프셋 정보만 초기화된 것이고, 데이터는 아직 Fetch 해 오지 않은 상태이다.
2. Logical Plan(사용자가 작성한 쿼리 혹은 코드)에 데이터 소스 정보를 덧붙인다.
3. Plan 내에서 사용되는 날짜/시간 관련 값들을 Wiring 한다.
4. Plan을 Dataset으로 변환한다.
5. 쿼리를 수행하고 결과를 Sink로 내보낸다. 이 때 필요한 경우 Commit 도 수행한다.

#### 내부 상태 정리

하나의 배치를 완료하였고, 다음 배치 실행 전 아래와 같이 내부 상태를 정리한다.

```
if (isCurrentBatchConstructed) {
  currentBatchId += 1
  isCurrentBatchConstructed = false
} else 
  Thread.sleep(pollingDelayMs)
}
```

이번 배치를 수행한 경우에는 `currentBatchId`를 1 증가시키고, `isCurrentBatchConstructured` 값을 `false`로 설정하므로써 다음 실행 시 새로 처리할 데이터 정보를 Fetch 해야한다는 사실을 Marking한다.

이번 배치를 수행하지 않을 경우에는 `pollingDelayMs` 만큼 Sleep 한 뒤 다음 Loop를 실행하게 된다.

# 정리

처음에는 Spark Structured Streaming에서 Kafka의 시작/종료 Offset을 어떻게 가져오나 정도만 간략히 알아보려는 의도였는데, Streaming Query 실행에 대한 부분이 다른 코드(Spark SQL 등을 처리하는)보다 간단해서 길게 정리하게 되었습니닫.

원본 글을 참고하긴 했지만 전체적 맥락을 파악하는 수준에서만 참고하였고, 대부분은 코드를 따라가면서 정리했기 때문에 정확하지 않거나 건너뛴 부분이 존재할 수 있습니다.

다만 이 글을 읽으시는 분들이 Spark Structured Streaming의 실행 순서 정도나 대략적인 동작 원리에 대해서는 알아가시지 않을까 하는 생각입니다. 도움이 되셨으면 좋겠습니다.