---
layout: post
title:  "Spark RDD의 count()는 어떻게 동작하는가?(Shuffle이 없는, Driver 편)"
date:   2019-07-21 21:00:00 +0900
author: leeyh0216
tags:
- apache-spark
---

# Spark RDD의 count()는 어떻게 동작하는가?(Shuffle이 없는, Driver 편)

Spark RDD의 기본 연산 중 하나인 count()가 어떻게 동작하는지 알아보도록 한다. 

Shuffle이 들어가면 분석이 너무 어렵기 때문에 Shuffle이 발생하지 않는 코드로만 추적해보았으며, 이번 글에서는 Driver에서 발생하는 과정만을 다룬다.

## Shuffle이 없는 count()

Spark 코드를 확인해보면 Shuffle이 존재하는 연산보다 Shuffle이 존재하지 않는 연산이 훨씬 쉽고, 읽어야 할 코드의 양이 적기 때문에 Shuffle이 존재하지 않는 RDD의 count()부터 알아보기로 한다.

아래와 같이 한 번의 map으로 이루어진 RDD의 count()가 어떻게 동작하는지 알아본다.

```
val parallelizedRDD = sc.parallelize(Seq(1,2,3,4,5,6,7,8,9,10), 2)
val transformedRDD = parallelizedRDD.map(_ + "th value")
transformedRDD.count()
```

### val parallelizedRDD = sc.parallelize(Seq(1,2,3,4,5,6,7,8,9,10), 2)

`SparkContext` 클래스에 있는 `parallelize`는 Caller(Driver)에 존재하는 Scala Collection을 RDD로 만드는 함수이다. 반환 타입은 `ParallelCollectionRDD`이다.

```
def parallelize[T: ClassTag](
    seq: Seq[T],
    numSlices: Int = defaultParallelism): RDD[T] = withScope {
  assertNotStopped()
  new ParallelCollectionRDD[T](this, seq, numSlices, Map[Int, Seq[String]]()
}
```

`ParallelCollectionRDD`는 `RDD`클래스를 상속하는데, 생성자를 눈여겨 보아야 한다.

```
private[spark] class ParallelCollectionRDD[T: ClassTag](
    sc: SparkContext,
    @transient private val data: Seq[T],
    numSlices: Int,
    locationPrefs: Map[Int, Seq[String]])
    extends RDD[T](sc, Nil) {
```

위 코드에서 호출하고 있는 부모 클래스 `RDD`의 생성자 코드는 아래와 같다.

```
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging {
```
`parallelize` 함수를 통해 생성되는 RDD는 부모 RDD가 존재하지 않기 때문에 Dependency가 존재하지 않아 RDD의 Dependency가 `null`로 만들어진다.

그리고 RDD에서 중요한 요소 중 하나인 Partition이 어떻게 생성되는지 확인해보도록 한다. 아래 코드는 `ParallelCollectionRDD` 클래스의 `getPartitions` 함수이다.

```
override def getPartitions: Array[Partition] = {
  val slices = ParallelCollectionRDD.slice(data, numSlices).toArray
  slices.indices.map(i => new ParallelCollectionPartition(id, i, slices(i))).toArray
}
```
`parallelize` 함수의 인자인 data를 numSlices개로 잘라서 ParallelCollectionPartition으로 만든다.

아래 코드를 실행해보면 생성된 파티션의 정보를 확인할 수 있다.

```
val parallelizedRDD = sc.parallelize(Seq(1,2,3,4,5,6,7,8,9,10), 2)
parallelizedRDD.partitions.map(_.asInstanceOf[ParallelCollectionPartition[Int]]).foreach(p => println(s"RDD: ${p.rddId}, Partition: ${p.index}, Values: ${p.values.mkString(",")}"))
```
출력 결과:
```
RDD: 0, Partition: 0, Values: 1,2,3,4,5
RDD: 0, Partition: 1, Values: 6,7,8,9,10
```

#### RDD의 ID

RDD의 ID는 RDD 객체 생성 시 `SparkContext`의 `newRddId` 함수를 호출하여 생성한다. 즉, RDD의 ID는 `SparkContext` 내에서 유일한 값이다.

**RDD 클래스의 id 필드**
```
/** A unique ID for this RDD (within its SparkContext). */
  val id: Int = sc.newRddId()
```
**SparkContext 클래스의 newRddId 함수**
```
/** Register a new RDD, returning its RDD ID */
  private[spark] def newRddId(): Int = nextRddId.getAndIncrement()
```

### val transformedRDD = parallelizedRDD.map(_ + "th value")

`parallelize` 함수를 통해 생성된 RDD에 `map` 함수를 적용하여 내부 값들을 변환하는 동작이다.

`map` T 타입(원본 RDD의 요소 타입)을 U 타입으로 변환하는 함수를 인자로 받고, `MapPartitionsRDD` 타입의 객체를 반환하는 함수이다.

`MapPartitionsRDD` 클래스의 생성자는 다음과 같다. 

```
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev) {
```
이 떄 주의깊게 보아야 할 점은 RDD의 생성자에 Caller RDD(예시에서는 ParallelizedRDD)를 넘기고 있다.

해당 생성자의 코드는 아래와 같다.

```
/** Construct an RDD with just a one-to-one dependency on one parent */
def this(@transient oneParent: RDD[_]) =
  this(oneParent.context, List(new OneToOneDependency(oneParent)))
```
아까와 달리 Dependency에 `null`이 아닌 `OneToOneDependency`를 넣어주고 있다. `map`은 `NarrowDependency`, 즉 Shuffle이 발생하지 않는 Transformation 연산이기 때문에 이전 RDD의 파티션과 새로 생성되는 RDD의 파티션이 1:1 관계를 가지게 되어 있다.

#### OneToOneDependency

Parent RDD와 Child RDD가 1:1 관계를 가지는 것을 표현하는 클래스이다.

```
/**
 * :: DeveloperApi ::
 * Represents a one-to-one dependency between partitions of the parent and child RDDs.
 */
@DeveloperApi
class OneToOneDependency[T](rdd: RDD[T]) extends NarrowDependency[T](rdd) {
  override def getParents(partitionId: Int): List[Int] = List(partitionId)
}
```

`parallelizedRDD`와 `transformedRDD`의 관계는 `OneToOneDependency`를 이용하여 아래와 같이 표현할 수 있다.

Parent(parallelizedRDD) <--- OneToOneDependency ---> Child(transformedRDD)

#### transformedRDD의 Partition 구성

`transformedRDD`의 파티션은 어떤 형태를 가지고 있을까? `transformedRDD`의 `getPartitions` 함수는 다음과 같다.

```
override def getPartitions: Array[Partition] = firstParent[T].partitions
```

부모 파티션의 Partition을 그대로 반환하게 되어 있다. map 과정에서 Shuffle이 발생하지 않기 때문에 RDD의 관계와 같이 Partition의 관계도 부모와 자식이 1:1로 유지되는 것을 알 수 있다.

코드를 통해 실제로도 동일한 파티션인지 확인해본다. 실제 값을 출력하는 방법과 객체를 비교하는 방법으로 검증해보았다.

**실제 값을 출력하는 방법(위의 parallelizedRDD에 대해 호출한 출력값과 동일해야 함)**
```
transformedRDD.partitions.map(_.asInstanceOf[ParallelCollectionPartition[Int]]).foreach(p => println(s"RDD: ${p.rddId}, Partition: ${p.index}, Values: ${p.values.mkString(",")}"))
```
출력 결과:
```
RDD: 0, Partition: 0, Values: 1,2,3,4,5
RDD: 0, Partition: 1, Values: 6,7,8,9,10
```
**partitions 객체가 동일한지 확인하는 방법**
```
assert(parallelizedRDD.partitions === transformedRDD.partitions)
```

### transformedRDD.count()

`count()`은 Action으로써 실제로 작업이 실행되는 함수이다. `count`의 코드는 아래와 같다.

```
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
```

`SparkContext` 객체의 `runJob` 함수를 호출한다. 호출되는 `runJob`은 내부적으로 다시 오버로딩 된 `runJob` 함수들을 호출하게 된다.

```
def runJob[T, U: ClassTag](rdd: RDD[T], func: Iterator[T] => U): Array[U] = {
    runJob(rdd, func, 0 until rdd.partitions.length)
  }
```
```
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: Iterator[T] => U,
      partitions: Seq[Int]): Array[U] = {
    val cleanedFunc = clean(func)
    runJob(rdd, (ctx: TaskContext, it: Iterator[T]) => cleanedFunc(it), partitions)
  }
```
```
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int]): Array[U] = {
    val results = new Array[U](partitions.size)
    runJob[T, U](rdd, func, partitions, (index, res) => results(index) = res)
    results
  }
```
```
def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```
대부분의 runJob의 매개변수의 구성은 아래와 같다.
* rdd: Action을 호출하는 RDD이다.
* func: rdd의 각 파티션에 적용할 함수이다.
* partitions: rdd를 구성하는 Partition의 Sequence

다른 코드(Clousure를 닫아주는 clean 등)들은 부가적인 부분이고 실제 Job을 수행하는 부분은 `dagScheduler`의 `runJob` 함수 호출이다.

### DagScheduler의 동작(1)

`count()`, `show()`, `save()` 등과 같은 Action을 호출하게 되면, DagScheduler에서 RDD를 Job으로 변환하여 실제 연산을 수행하게 된다.

`SparkContext`에서 호출한 `DagScheduler`의 `runJob`의 코드는 아래와 같다.

```
def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
  val start = System.nanoTime
  val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
  ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
  waiter.completionFuture.value.get match {
    case scala.util.Success(_) =>
      logInfo("Job %d finished: %s, took %f s".format
        (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
    case scala.util.Failure(exception) =>
      logInfo("Job %d failed: %s, took %f s".format
        (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
      val callerStackTrace = Thread.currentThread().getStackTrace.tail
      exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
      throw exception
  }
}
```
실제 Job을 submit하고 기다리는 부분은 아래 코드이다.
```
val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
```

다시 `submitJob` 코드를 확인해보면 다음과 같다.
```
def submitJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): JobWaiter[U] = {
  // Check to make sure we are not launching a task on a partition that does not exist.
  val maxPartitions = rdd.partitions.length
  partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
    throw new IllegalArgumentException(
      "Attempting to access a non-existent partition: " + p + ". " +
      "Total number of partitions: " + maxPartitions)
  }

  val jobId = nextJobId.getAndIncrement()
  if (partitions.isEmpty) {
    val time = clock.getTimeMillis()
    listenerBus.post(
      SparkListenerJobStart(jobId, time, Seq[StageInfo](), properties))
    listenerBus.post(
      SparkListenerJobEnd(jobId, time, JobSucceeded))
    // Return immediately if the job is running 0 tasks
    return new JobWaiter[U](this, jobId, 0, resultHandler)
  }

  assert(partitions.nonEmpty)
  val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
  val waiter = new JobWaiter[U](this, jobId, partitions.size, resultHandler)
  eventProcessLoop.post(JobSubmitted(
    jobId, rdd, func2, partitions.toArray, callSite, waiter,
    SerializationUtils.clone(properties)))
  waiter
}
```

코드의 윗부분들은 Partition이 없는 경우 등 특수한 상황을 처리하는 부분이고, 우리가 실제로 눈여겨보아야 할 부분은 아래의 코드이다.

```
val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
val waiter = new JobWaiter[U](this, jobId, partitions.size, resultHandler)
eventProcessLoop.post(JobSubmitted(
  jobId, rdd, func2, partitions.toArray, callSite, waiter,
  SerializationUtils.clone(properties)))
waiter
```

`EventLoop` 타입인 `eventProcessLoop` 객체의 `post` 함수 호출을 통해 실제 작업을 호출하게 된다.

### EventLoop와 DAGSchedulerEventProcessLoop

`EventLoop` Job을 수행하는 Action 연산이나 Job을 종료하는 Job Cancel이 호출될 경우 이러한 작업을 처리하는 Thread를 만들어 수행하는 추상 클래스이며, `DagSchedulerEventProcessLoop`는 `EventLoop`를 상속한 구체 클래스이다.

위의 `DagScheduler`에서 `post`를 호출하게 되면 EventLoop의 eventQueue에 작업을 넣는다.

```
def post(event: E): Unit = {
  if (!stopped.get) {
    if (eventThread.isAlive) {
      eventQueue.put(event)
    } else {
      onError(new IllegalStateException(s"$name has already been stopped accidentally."))
    }
  }
}
```
`EventLoop`는 무한루프를 돌며 EventQueue에서 작업을 꺼내 `onReceive` 함수를 호출한다.
```
override def run(): Unit = {
  try {
    while (!stopped.get) {
      val event = eventQueue.take()
      try {
        onReceive(event)
      } catch {
        case NonFatal(e) =>
          try {
            onError(e)
          } catch {
            case NonFatal(e) => logError("Unexpected error in " + name, e)
          }
      }
    }
  } catch {
    case ie: InterruptedException => // exit even if eventQueue is not empty
    case NonFatal(e) => logError("Unexpected error in " + name, e)
  }
}
```

`onReceive` 함수는 `EventLoop` 클래스의 구현체인 `DagSchedulerEventProcessLoop`의 `onReceive`에 구현되어 있다.

```
override def onReceive(event: DAGSchedulerEvent): Unit = {
  val timerContext = timer.time()
  try {
    doOnReceive(event)
  } finally {
    timerContext.stop()
  }
}
```

`onReceive`의 내부에서는 다시 `doOnReceive` 함수를 호출하게 된다. `doOnReceive` 함수에서는 매개변수로 전달받은 Event를 패턴매칭을 통해 걸러내고 `DagScheduler`의 handle 함수에게 넘기게 된다.

```
private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
  case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
    dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)

  case MapStageSubmitted(jobId, dependency, callSite, listener, properties) =>
    dagScheduler.handleMapStageSubmitted(jobId, dependency, callSite, listener, properties)

  case StageCancelled(stageId, reason) =>
    dagScheduler.handleStageCancellation(stageId, reason)

  case JobCancelled(jobId, reason) =>
    dagScheduler.handleJobCancellation(jobId, reason)
...생략
```

우리가 DagScheduler에서 호출한 submitJob 코드를 보면 아래와 같았다.

```
eventProcessLoop.post(JobSubmitted(
  jobId, rdd, func2, partitions.toArray, callSite, waiter,
  SerializationUtils.clone(properties)))
```

`JobSumitted` 타입의 객체를 전달했기 때문에 case문의 첫번째에 걸려 `DagScheduler`의 `handleJobSubmitted`를 호출한다.
```
case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
  dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)
```

`handleJobSubmitted` 함수가 워낙 길어 필요한 부분만 간추리면 아래와 같이 볼 수 있다.

```
private[scheduler] def handleJobSubmitted(jobId: Int,
    finalRDD: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    callSite: CallSite,
    listener: JobListener,
    properties: Properties) {
  var finalStage: ResultStage = null
  try {
    // New stage creation may throw an exception if, for example, jobs are run on a
    // HadoopRDD whose underlying HDFS files have been deleted.
    finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
  } catch {
    ...생략
  }
  // Job submitted, clear internal data.
  barrierJobIdToNumTasksCheckFailures.remove(jobId)

  val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
  clearCacheLocs()
  ...생략
  jobIdToActiveJob(jobId) = job
  activeJobs += job
  finalStage.setActiveJob(job)
  val stageIds = jobIdToStageIds(jobId).toArray
  val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
  listenerBus.post(
    SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
  submitStage(finalStage)
}
```

`createResultStage` 함수를 호출하여 Action 연산을 호출한 RDD를 통해 ResultStage를 생성하고 이를 `submitStage` 함수를 통해 다시 DagSchedulerEventProcessLoop에게 전달한다.

#### Stage의 정의와 종류

Stage는 동일한 Shuffle Dependency를 가진 Task들의 집합이다(즉, Shuffle이 Stage를 나누는 기준이 된다). Stage를 구성하는 Task들은 Narrow Dependency를 통해 연결되어 있으며, 병렬로 실행될 수 있다.

Stage는 `ShuffleMapStage`와 `ResultStage`로 나뉘어진다.

* ShuffleMapStage: Task들의 결과가 다른 Stage의 Input으로 들어가는 Stage
* ResultStage: Action의 결과를 출력하기 위해 동작하는 Stage

즉, 1개의 Spark Job은 1개 이상의 ShuffleMapStage와 1개의 ResultStage로 구성된다고 볼 수 있다.

### DagScheduler의 동작(2)

`createResultStage` 함수의 코드는 아래와 같다.

```
private def createResultStage(
    rdd: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    jobId: Int,
    callSite: CallSite): ResultStage = {
  checkBarrierStageWithDynamicAllocation(rdd)
  checkBarrierStageWithNumSlots(rdd)
  checkBarrierStageWithRDDChainPattern(rdd, partitions.toSet.size)
  val parents = getOrCreateParentStages(rdd, jobId)
  val id = nextStageId.getAndIncrement()
  val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite)
  stageIdToStage(id) = stage
  updateJobIdStageIdMaps(jobId, stage)
  stage
}
```

`getOrCreateParentStages` 라는 함수를 통해 ResultStage 를 생성하는 이전 Stage를 가져온다.

여기서 우리가 처음 호출했던 코드를 다시 확인해보자.

```
val parallelizedRDD = sc.parallelize(Seq(1,2,3,4,5,6,7,8,9,10), 2)
val transformedRDD = parallelizedRDD.map(_ + "th value")
transformedRDD.count()
```

`parallelize`와 `map`으로만 구성된 RDD이기 때문에 Shuffle이 존재하지 않고, 결과적으로 이 코드는 1개의 Stage로만 이루어져 있다.

따라서 위의 `getOrCreateParentStages` 함수는 `null`을 반환하게 된다. 따라서 `createResultStage`에서는 1개의 ResultStage만 반환하게 된다.

생성된 Stage는 `handleJobSubmitted`의 `submitStage` 함수를 통해 처리된다.

```
private[scheduler] def handleJobSubmitted(jobId: Int,
    finalRDD: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    callSite: CallSite,
    listener: JobListener,
    properties: Properties) {
  var finalStage: ResultStage = null
  try {
    // New stage creation may throw an exception if, for example, jobs are run on a
    // HadoopRDD whose underlying HDFS files have been deleted.
    finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
  } catch {
    ...생략
  }
  ...생략
  submitStage(finalStage)
}
```

`submitStage`의 코드는 아래와 같다.

```
private def submitStage(stage: Stage) {
  val jobId = activeJobForStage(stage)
  if (jobId.isDefined) {
    logDebug("submitStage(" + stage + ")")
    if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
      val missing = getMissingParentStages(stage).sortBy(_.id)
      logDebug("missing: " + missing)
      if (missing.isEmpty) {
        logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
        submitMissingTasks(stage, jobId.get)
      } else {
        for (parent <- missing) {
          submitStage(parent)
        }
        waitingStages += stage
      }
    }
  } else {
    abortStage(stage, "No active job for stage " + stage.id, None)
  }
}
```
일단 `getMissingParentStages` 함수를 통해 해당 Stage 이전에 실행되어야 하는 Stage를 찾아내고 실행한다. 이 예제에서는 이전에 실행되어야 하는 Stage가 존재하지 않으므로 `submitMissingTasks` 함수가 실행된다.

`submitMissingTasks` 함수는 워낙 길어서 필요한 부분만 잘라보았는데, 중요한 부분은 아래와 같다.

```
val tasks: Seq[Task[_]] = try {
  val serializedTaskMetrics = closureSerializer.serialize(stage.latestInfo.taskMetrics).array()
  stage match {
    case stage: ShuffleMapStage =>
      stage.pendingPartitions.clear()
      partitionsToCompute.map { id =>
        val locs = taskIdToLocations(id)
        val part = partitions(id)
        stage.pendingPartitions += id
        new ShuffleMapTask(stage.id, stage.latestInfo.attemptNumber,
          taskBinary, part, locs, properties, serializedTaskMetrics, Option(jobId),
          Option(sc.applicationId), sc.applicationAttemptId, stage.rdd.isBarrier())
      }

    case stage: ResultStage =>
      partitionsToCompute.map { id =>
        val p: Int = stage.partitions(id)
        val part = partitions(p)
        val locs = taskIdToLocations(id)
        new ResultTask(stage.id, stage.latestInfo.attemptNumber,
          taskBinary, part, locs, id, properties, serializedTaskMetrics,
          Option(jobId), Option(sc.applicationId), sc.applicationAttemptId,
          stage.rdd.isBarrier())
      }
  }
} catch {
  case NonFatal(e) =>
    abortStage(stage, s"Task creation failed: $e\n${Utils.exceptionString(e)}", Some(e))
    runningStages -= stage
    return
}
```

위의 코드는 Stage를 Task로 분리하여 TaskSet 형태로 만드는 과정이다. Task는 1개의 Partition을 연산하는 작업을 의미하며, TaskSet은 1개 이상의 Task로 구성된 개념으로 사실상 Stage와 동일하다고 생각하면 될 것 같다.

Task는 Stage의 종류에 따라 `ShuffleMapStage`일 경우 `ShuffleMapTask`, `ResultStage`일 경우 `ResultTask`로 구분되어 생성된다.

이렇게 생성된 TaskSet은 `TaskScheduler`의 `submitTasks` 함수에 전달되어 실행된다.

```
taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptNumber, jobId, properties))
```

`TaskScheduler`의 구현체인 `TaskSchedulerImpl`의 `submitTasks` 에서는 다시 `SchedulerBackend`의 구현체인 `CoarseGrainedSchedulerBackend`의 `receiveOffers`를 호출하게 된다.

> 사실 이 부분에서 receiveOffers의 인자로 넘기는게 없는데, 실행시킬 수 있다는게 이해가 가지 않는다. 다만 `CoarseGrainedSchedulerBackend`에서 `TaskSchedulerImpl` 객체를 내부 변수로 가지고 있기 때문에 `TaskScehdulerImpl`의 내부 변수를 참조하여 실행시키는 것이 아닌가 하는 추측으로 진행을 했다.

### SchedulerBackend(CoarseGrainedSchedulerBackend)의 동작

`SchedulerBackend`의 구현체인 `CoarseGrainedSchedulerBackend`의 `receiveOffer`는 다시 `DriverEndpoint` 객체의 `send` 함수를 호출하여 실행을 요청한다.

`DriverEndpoint`의 `receive` 함수에서는 `ReceiveOffer` 이벤트를 수신하여 `makeOffers` 함수를 수행한다.

```
case ReviveOffers =>
  makeOffers()
```

`makeOffers` 함수에서는 실행할 Task의 정보를 받아와 `launchTask` 함수를 통해 실제 Task 실행을 수행한다.

```
private def makeOffers() {
  // Make sure no executor is killed while some task is launching on it
  val taskDescs = withLock {
    // Filter out executors under killing
    val activeExecutors = executorDataMap.filterKeys(executorIsAlive)
    val workOffers = activeExecutors.map {
      case (id, executorData) =>
        new WorkerOffer(id, executorData.executorHost, executorData.freeCores,
              Some(executorData.executorAddress.hostPort),
              executorData.resourcesInfo.map { case (rName, rInfo) =>
              (rName, rInfo.availableAddrs.toBuffer)}
        )
    }.toIndexedSeq
    scheduler.resourceOffers(workOffers)
  }
  if (taskDescs.nonEmpty) {
    launchTasks(taskDescs)
  }
}
```

> 위 함수에서 아마 TaskScheduler에서 만든 TaskSet을 받아 처리하는 것 같은데, 어디서 데이터를 가져오는지는 잘 모르겠다. 아마 executorDataMap일 것 같은데.. 좀 더 살펴봐야 할 것 같다.

대망의 `launchTask` 함수이다. TaskSet을 구성하는 Task들을 Serialization하여 각 Executor에게 실행을 요청한다.

```
private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
  for (task <- tasks.flatten) {
    val serializedTask = TaskDescription.encode(task)
    if (serializedTask.limit() >= maxRpcMessageSize) {
      Option(scheduler.taskIdToTaskSetManager.get(task.taskId)).foreach { taskSetMgr =>
        try {
          var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
            s"${RPC_MESSAGE_MAX_SIZE.key} (%d bytes). Consider increasing " +
            s"${RPC_MESSAGE_MAX_SIZE.key} or using broadcast variables for large values."
          msg = msg.format(task.taskId, task.index, serializedTask.limit(), maxRpcMessageSize)
          taskSetMgr.abort(msg)
        } catch {
          case e: Exception => logError("Exception in error callback", e)
        }
      }
    }
    else {
      val executorData = executorDataMap(task.executorId)
      // Do resources allocation here. The allocated resources will get released after the task
      // finishes.
      executorData.freeCores -= scheduler.CPUS_PER_TASK
      task.resources.foreach { case (rName, rInfo) =>
        assert(executorData.resourcesInfo.contains(rName))
        executorData.resourcesInfo(rName).acquire(rInfo.addresses)
      }

      logDebug(s"Launching task ${task.taskId} on executor id: ${task.executorId} hostname: " +
        s"${executorData.executorHost}.")

      executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
    }
  }
}
```

여기까지가 Driver 단에서 RDD의 Action을 수행한 후 Task의 실행을 Executor에게 Launch하도록 요청하는 과정이다.

3줄짜리 코드를 실행하기 위해 이렇게 많은 구성요소들이 결합하여 동작하는줄 모르고 있었고, 생각보다 실행이 매우 복잡하다는 것을 느낄 수 있었다.