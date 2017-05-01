---
layout: post
title:  "[Spark] Spark RDD"
date:   2017-04-30 16:50:00 +0900
author: leeyh0216
categories: dev lang spark
---

> 이 문서는 Spark의 RDD에 대해 서술한 문서입니다.

## SparkContext

[SparkContext Javadoc](https://spark.apache.org/docs/latest/api/java/org/apache/spark/SparkContext.html)에서 SparkContext는 다음과 같이 정의되어 있다.

> Spark 프로그램의 진입점. SparkContext는 Spark Cluster로의 연결을 의미하며 RDD, Accumulator, Broadcast 변수를 생성할 수 있다.
하나의 JVM에는 하나의 SparkContext만을 생성할 수 있으므로, 새로운 SparkConext를 생성하기 전에는 활성화 된 SparkConext의 stop() 메소드를 호출한 후 새로운 SparkContext를 생성할 수 있다.

SparkContext를 생성하기 위해서는 필수적으로 Application Name과 Master(위에서 언급한 SparkCluster로의 연결을 위한 정보) 정보를 설정해야 한다. JavaDoc에서는 다양한 방법으로 초기화 할 수 있다고 언급되어 있지만, 예제로는 SparkConf를 이용하여 생성하도록 한다.

{% highlight scala %}
import org.apache.spark.{SparkConf, SparkContext}

//Master는 local, AppName은 TestApplication으로 지정한 SparkConfiguration 객체를 생성한다.
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
{% endhighlight %}

위와 같이 SparkConf 객체의 setMaster, setAppName을 이용하여 SparkContext에 넣을 정보들을 설정할 수 있다.
스파크는 로컬 모드와 클러스터 모드로 동작할 수 있는데, 개발 시에는 로컬 모드를 사용할 것이므로 Master에 local을 설정해 준다. 이 때 꺾쇠 괄호를 이용하여 사용할 CPU 코어 갯수를 설정할 수 있는데, 듀얼 코어 프로세서를 사용하므로 2를 적어주면 된다.(클러스터 모드의 Executor Core와 동일한 요소)
AppName은 Spark UI(Default : http://localhost:4040)에서 해당 Application을 구분하기 위한 이름이다. 위 2개의 요소 이외에도 여러가지 정보를 설정할 수 있지만, 추후 작성할 Spark의 설정 Post에서 다루도록 한다.

## RDD

### RDD의 정의
[RDD Javadoc](https://spark.apache.org/docs/latest/api/java/org/apache/spark/rdd/RDD.html)에서 RDD는 다음과 같이 정의되어 있다.

> Resilient Distributed Dataset (RDD)는 변경이 불가능하고 병렬적으로 처리될 수 있는 파티셔닝 된 컬렉션이다. RDD는 map,filter,persist 등의 기능을 포함하고 있다. 더욱이 PairRDDFunction은 Key-Value 형태의 RDD에 대해 처리할 수 있는 groupByKey, join 등의 연산을 포함하고 있다. 내부적으로 각 RDD는 다섯가지 구성요소들로 이루어져 있다.
- Partition List
- 각 Split을 처리할 수 있는 Function
- 다른 RDD에 대한 Dependency
- 추가적으로 Key-Value RDD에 사용하는 Partitioner
- 추가적으로 각 Split이 처리될 수 있는 최적의 장소 

### RDD 생성하기

SparkContext를 생성하였으므로, RDD를 생성할 수 있다.
Spark에서는 두가지 방법으로 RDD를 생성할 수 있다.
첫번째 방법은 SparkContext의 parallelize() 메서드를 이용하여 Collection 객체를 RDD로 변환하는 방식이다.
parallelize 메서드의 정의는 다음과 같이 되어 있다.
{% highlight scala %}
public <T> RDD<T> parallelize(scala.collection.Seq<T> seq,
                     int numSlices,
                     scala.reflect.ClassTag<T> evidence$1)
{% endhighlight %}
Scala Collection을 분산하여 RDD의 형태로 만든다고 되어 있는데, 일단 Scala Collection들이 상속 받는 Seq와 numSlices라는 값이 매개 변수 목록에 있다.
Seq의 경우 위에서 말했던 것과 같이 RDD로 만들 Scala Collection이고 numSlice는 해당 Collection을 몇 개의 Partition으로 분할할 것인지에 대한 값이다(넣지 않는다면 Spark의 DefaultParalleism 값을 따른다고 되어 있다).

매우 주의해야 할 점은, Seq에 들어가는 값은 Immutable을 권장한다고 되어 있다. parallelize 함수는 Lazy Function이기 때문에 Action 함수가 호출되기 전까지는 RDD로 변환되지 않는데, parallelize 함수에 Mutable Collection 객체를 넣은 뒤 Action이 있기 전에 Mutable Collection을 변경하면 RDD도 변경된 Mutable Collection을 이용하여 생성되기 때문에 헷갈릴 수 있다는 것이다.

또한 빈 Seq를 사용하면 비어 있는 RDD를 생성할 수 있다고 적혀 있다.

위 사항들을 고려하여 3가지 예제를 실행해 보았다.

- Partition 갯수와 기본 출력 확인
{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("a","b","c"))
println("Num Partitions : "+rdd.getNumPartitions)
rdd.foreach{ s: String => println(String.format("value : %s",s))}

/*출력 결과
   Num Partitions : 2
   value : b
   value : c
   value : a
*/
{% endhighlight %}
parallelize 함수 호출 시 numSlices를 설정하지 않았기 때문에 DefaultParalleism(2)를 사용하였고, Collection에 있던 3개의 값이 출력되는 것을 확인할 수 있다. 단, RDD의 sort 등의 메소드를 사용하지 않는 이상 순서를 보장하지 않기 때문에 출력 순서가 a->b->c 는 아닌 것을 확인할 수 있다.

- Partition 갯수를 설정해보기
{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("a","b","c"),3)
println("Num Partitions : "+rdd.getNumPartitions)
rdd.foreach{ s: String => println(String.format("value : %s",s))}

/*출력 결과
  Num Partitions : 3
  value : b
  value : a
  value : c
*/
{% endhighlight %}
parallelize 함수의 numSlices를 3으로 설정해 주니 getNumPartitions가 3으로 출력되는 것을 확인할 수 있었다.
RDD에는 foreach만이 아닌 foreachPartitions라는 함수도 있는데, 이 함수를 이용하면 각 Partition에 어떤 값들이 존재하는지 확인할 수 있다.

- Partition에 어떤 값이 있는지 확인해보기
{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("a","b","c"),5)
println("Num Partitions : "+rdd.getNumPartitions)
rdd.foreachPartition{ sList =>
   println("Partition 구분 용도 : "+UUID.randomUUID().toString)
   sList.foreach{ s=> println("value : "+s)}
}

/*출력결과
Num Partitions : 5
Partition 구분 용도 : 1ebeb279-d452-4038-8177-57e2bf65c273
Partition 구분 용도 : 07c3b91f-b39a-4629-af24-0df40c18a76b
value : a
Partition 구분 용도 : 89b776d0-e907-49d9-a6a9-36978335aa10
Partition 구분 용도 : 21a8bcff-c8f7-4bbe-8e34-7686a97dfee5
value : c
Partition 구분 용도 : 7d48e0b4-003a-4ad9-9ffe-866103fab6a9
value : b
*/
{% endhighlight %}
위와 같이 5개의 Partition이 생성되는 것을 확인할 수 있고, 일부 Partition에는 값이 존재하는 것을 확인할 수 있다(값은 3개인데 Partition이 5개이기 때문에 최소 2개는 빈 Partition일 수밖에 없다)

### RDD의 트랜스포메이션 연산

트랜스포메이션은 기존 RDD를 이용하여 새로운 RDD를 생성하는 연산이다.

#### map
하나의 입력을 받아 하나의 결과를 돌라주는 함수를 인자로 받는 연산.
map 함수는 RDD의 모든 요소들에 대해 위 함수를 적용한 뒤 새로운 RDD를 생성한다.

아래는 1,2,3으로 이루어진 List를 RDD로 변환한 뒤 해당 RDD의 각 요소들에 1을 더하여 새로운 RDD를 생성하는 예제이다.
{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val sampleRDD = sc.parallelize(List(1,2,3))
val onePlusRDD = sampleRDD.map{ i => i+1}
onePlusRDD.foreach{ r => println(s"result : $r")}

/*출력결과
result : 3
result : 2
result : 4
*/
{% endhighlight %}

위에서는 map 함수에 사용되는 함수가 Int => Int 형태였지만, 꼭 Input Type과 Output Type이 일치할 필요는 없다.
예를 들어 다음은 String형태의 RDD의 각 요소들의 길이로 이루어진 Int 형태의 요소들로 이루어진 RDD를 생성하는 예제이다.

{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("hello","my","name","is","rdd"))
rdd.map{ i => i.length}.foreach{ len => println(s"Word's length : $len")}

/*출력결과
Word's length : 5
Word's length : 2
Word's length : 4
Word's length : 2
Word's length : 3
*/
{% endhighlight %}

반드시 익명 함수를 전달할 필요 없이, class나 object의 Lambda Function을 값으로 전달할 수 있다(Scala의 함수는 1급 객체이기 때문에 이런 식으로 전달 하는 것이 가능하다, Java에서는 함수가 1급객체가 아니기 때문에 Java7 까지는 Interface를 전달했으며, Serialize가 가능해야 했다).

{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val calcWordLength = { w : String => w.length}
val rdd = sc.parallelize(List("hello","my","name","is","rdd"))
rdd.map(calcWordLength).foreach{ len => println(s"Word's length : $len")}

/*출력결과
Word's length : 4
Word's length : 2
Word's length : 3
Word's length : 5
Word's length : 2
*/
{% endhighlight %}

#### flatMap
map의 경우 1개의 Input에 대해 1개의 Output을 생성하는 함수를 인자로 가졌지만, flatMap의 경우 1개의 Input에 대해 0개 이상의 Output(TraversableOnce)을 생성하는 함수를 인자로 가진다.
단, flatMap 형태로 생성된 RDD는 RDD[TraversableOnce[U]] 형태가 아닌 RDD[U] 형태로 변경된다. 즉, List 안에 있는 요소들을 풀어서 RDD를 생성하게 된다.

다음은 , Delimeter로 이루어진 Word를 Split하여 단일 Word 형태의 RDD로 변환하는 예제이다.

{% highlight scala %}
val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("john,maria,denis","coke,fanta,cider","seoul,incheon,kyungki"))
rdd.flatMap{ s => s.split(",")}.foreach(w => println(w))

/*출력결과
john
maria
denis
coke
fanta
cider
seoul
incheon
kyungki
*/
{% endhighlight %}

#### mapPartitions
우리는 RDD가 1개의 거대한 Collection을 파티셔닝하여 가지고 있는 것과 각 Partition 또한 Collection을 가지고 있다는 것을 알고 있지만, map의 경우 우리가 이러한 추상적인 개념을 생각하지 않고 RDD를 단순히 거대한 Collection 이라고 생각하게 만들어 쉽게 접근할 수 있게 한다. 하지만, DB연결 등의 작업을 수행할 때 각 map 함수 내에서 연결을 하면, DB 서버 등에 엄청난 부하가 발생하게 될 것이다.
따라서 Spark는 Partition 단위로 연산을 수행할 수 있는 mapPartition을 제공한다. 각 Partition은 Iterable 형태의 Collection을 1개씩 가지고 있기 때문에, mapPartition 형태의 함수의 입력은 Traversable[U]이다. 
아래는 flatmap예제에서 생성한 RDD[String]의 길이를 파티션별로 계산하는 예제이다.

{% highlight scala %}
 val conf = new SparkConf().setMaster("local[2]").setAppName("TestApplication")
val sc = new SparkContext(conf)
val rdd = sc.parallelize(List("john,maria,denis","coke,fanta,cider","seoul,incheon,kyungki"))
rdd.flatMap{ wl => wl.split(",")}.repartition(3).mapPartitions{ iter => println("DB연결"); iter.map{i => i.length}}.foreach(s => println(s))

/*출력결과
DB연결
DB연결
4
5
4
5
5
7
DB연결
5
5
7
*/
{% endhighlight %}

3개의 파티션으로 RDD를 분할한 뒤 mapPartitions 함수의 처음에 DB연결이라는 문구를 출력했다. 결과적으로 각 Partition을 계산할 때 1회씩 DB연결이라는 문구를 출력하였고, 각 Partition을 이루는 iterable[String]에 대해 map함수를 적용하여 iterable[Int] 형태의 결과를 리턴한 후 출력하게 하였다.
