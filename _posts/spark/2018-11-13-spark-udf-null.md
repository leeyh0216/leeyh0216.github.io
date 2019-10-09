---
layout: post
title:  "Spark UDF와 DataSet에서의 NULL 처리"
date:   2018-11-13 10:00:00 +0900
author: leeyh0216
tags:
- spark
---

# 개요

Spark SQL에서는 UDF(User Defined Function)를 만들 수 있는 기능을 제공한다.

SQL만으로 처리가 힘들거나 코드가 지저분해지는 상황이 발생했을 때 유용하게 사용할 수 있다.

다만 NULL 처리에 관해서는 매우 신경을 써 줘야하는데, 오늘 1시간 넘게 UDF 구현 시 NULL 관련 오류를 접했던 삽질을 정리한다.

# UDF에서의 NULL 처리

## String Type

String 타입의 경우 애초에 Reference 타입이기 때문에, UDF에서의 NULL 처리가 간결하다.

주어진 문자열을 시작 위치부터 2만큼 잘라내는 UDF를 만들어보자.

{% highlight scala %}
//UDF 생성 및 등록
val subStrUDF = udf { s: String => s.substring(0,2) }
spark.sqlContext.udf.register("subStrUDF", subStrUDF)

//UDF 테스트
spark.sql("""SELECT "hello" AS a""").createOrReplaceTempView("tbl")
spark.sql("""SELECT subStr(a) FROM tbl""").show(1,false)

+------+
|result|
+------+
|he    |
+------+
{% endhighlight %}

정상적으로 처리되는 것을 확인할 수 있다. 이제 NULL 값도 추가하여 테스트를 진행해보자.

{% highlight scala %}
//UDF 생성 및 등록
val subStrUDF = udf { s: String => s.substring(0,2) }
spark.sqlContext.udf.register("subStrUDF", subStrUDF)

//UDF 테스트
Seq("hello", null).toDF("a").createOrReplaceTempView("tbl")
spark.sql("""SELECT subStr(a) FROM tbl""").show(2,false)

org.apache.spark.SparkException: Failed to execute user defined function($anonfun$1: (string) => string)
  at org.apache.spark.sql.catalyst.expressions.ScalaUDF.eval(ScalaUDF.scala:1058)
  at org.apache.spark.sql.catalyst.expressions.Alias.eval(namedExpressions.scala:139)
  ...생략
Caused by: java.lang.NullPointerException
  at $anonfun$1.apply(<console>:23)
  at $anonfun$1.apply(<console>:23)
  ...생략
{% endhighlight %}

위와 같이 NullPointerException이 발생하는 것을 확인할 수 있다.

이 경우는 UDF에서 인자 s가 NULL인 경우에 대한 예외처리만 해주면 된다.

{% highlight scala %}
//UDF 생성 및 등록
val subStrUDF = udf { s: String => if(s == null) "" else s.substring(0,2) }
spark.sqlContext.udf.register("subStrUDF", subStrUDF)

//UDF 테스트
Seq("hello", null).toDF("a").createOrReplaceTempView("tbl")
spark.sql("""SELECT subStr(a) FROM tbl""").show(2,false)

/****************
    실행 결과
    +------+
    |UDF(a)|
    +------+
    |he    |
    |      |
    +------+
*****************/
{% endhighlight %}

위와 같이 일반적인 NULL 처리 방식으로 쉽게 구현이 가능한 것을 확인할 수 있다.

## Int, Long 등의 숫자 Primitive Type

오늘 삽질의 원인이 되었던 Integer, Long 타입이다.

처음에는 처리하려던 필드가 null이 발생할 수 있는 필드인지 몰랐기 때문에, UDF 인자를 모두 Primitive Type인 Int와 Long 등으로 정의했다.

실제 회사 코드를 가져올 수는 없으니, Int 타입의 값을 받아 1 증가시켜 반환하는 incrUDF 라는 UDF를 정의한 후 테스트해보도록 하자.

{% highlight scala %}
//UDF 생성 및 등록
val incrUDF = udf { a: Int => a+1 }
spark.sqlContext.udf.register("incrUDF", incrUDF)

//UDF 테스트
spark.sql("""select 1 as a""").createOrReplaceTempView("tbl")
spark.sql("""select incrUDF(a) from tbl""").show(1,false)

/****************
+------+
|UDF(a)|
+------+
|2     |
+------+
*****************/
{% endhighlight %}

정상적으로 1 값에 1을 더해 2를 반환하여 결과가 2로 나타난 것을 볼 수 있다.

그렇다면 a 필드에 null 값을 넣어보면 어떨까?

{% highlight scala %}
//UDF 생성 및 등록
val incrUDF = udf { a: Int => a+1 }
spark.sqlContext.udf.register("incrUDF", incrUDF)

//UDF 테스트
spark.sql("""select cast(null as int) as a""").createOrReplaceTempView("tbl")
spark.sql("""select incrUDF(a) from tbl""").show(1,false)

/****************
+------+
|UDF(a)|
+------+
|null  |
+------+
*****************/
{% endhighlight %}

null이 반환된 것을 볼 수 있다. 사실 위 코드에서 의도했던 결과는 1이었다.

이유는 아래와 같은 코드를 작성해보면 알 수 있다.

{% highlight scala %}
scala> val tmp = null.asInstanceOf[Int] + 1
tmp: Int = 1
{% endhighlight %}

Scala의 null을 asInstanceOf 메소드를 이용하여 캐스팅해보면 Int의 기본 값인 0이 되는 것을 확인할 수 있다.

당연히 Spark이 Scala 위에서 구현되었기 때문에 UDF도 언어적인 측면을 따라갈 것이라 생각했지만, SQL 내에서 실행되는 함수이기 때문에 NULL 처리 또한 SQL을 따라가고 있었다.

이러한 문제를 피하기 위해서는 Primitive Type이 아닌 Object Type을 사용하면 된다.

위의 코드를 아래와 같이 변경하여 테스트해보면 정상적으로 동작하는 것을 확인할 수 있다.

{% highlight scala %}
//UDF 생성 및 등록
val incrUDF = udf { a: java.lang.Integer => if(a == null) 1 else a.intValue() + 1 }
spark.sqlContext.udf.register("incrUDF", incrUDF)

//UDF 테스트
spark.sql("""select cast(null as int) as a""").createOrReplaceTempView("tbl")
spark.sql("""select incrUDF(a) from tbl""").show(1,false)

/****************
+------+
|UDF(a)|
+------+
|1     |
+------+
*****************/
{% endhighlight %}

의도한 대로 동작하는 것을 확인할 수 있다.

# DataSet 사용 시의 null 처리

만일 Dataframe을 Case Class에 매핑시켜 Dataset으로 만들었을 때는 각 타입들이 어떻게 동작할까?

아래와 같은 코드를 이용하여 테스트 해 보았다.

{% highlight scala %}
case class A(a: Int, b: Long, c: String)

spark.sql("select cast(null as int) as a, cast(null as long) as b, cast(null as string) as c").as[A].map(r => A(r.a + 1, r.b + 1, r.c + " is string"))

Driver stacktrace:
  at org.apache.spark.scheduler.DAGScheduler.org$apache$spark$scheduler$DAGScheduler$$failJobAndIndependentStages(DAGScheduler.scala:1602)
  at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1590)
  at org.apache.spark.scheduler.DAGScheduler$$anonfun$abortStage$1.apply(DAGScheduler.scala:1589)
  ... 생략
Caused by: java.lang.NullPointerException: Null value appeared in non-nullable field:
- field (class: "scala.Int", name: "a")
- root class: "A"
If the schema is inferred from a Scala tuple/case class, or a Java bean, please try to use scala.Option[_] or other nullable types (e.g. java.lang.Integer instead of int/scala.Int).
... 생략
{% endhighlight %}

위와 같이 오류가 발생하는 것을 확인할 수 있고, non-nullable 필드에 null이 발생하였으니, 해당 필드를 Option으로 감싸주라는 제안이 나온다.

그렇다면 A 클래스의 Primitive Type인 a(Int)와 b(Long)을 Option으로 감싸서 처리해보자.

{% highlight scala %}
case class A(a: Int, b: Long, c: String)

spark.sql("""select cast(null as int) as a, cast(null as long) as b, cast(null as string) as c""").as[A].map(r => A(Some(r.a.getOrElse(0) + 1), Some((r.b.getOrElse(0L) + 1L)), r.c + " is string")).toDF().show(1,false)

+---+---+--------------+
|a  |b  |c             |
+---+---+--------------+
|1  |1  |null is string|
+---+---+--------------+
{% endhighlight %}

위와 같이 a, b가 null일 때는 정상적으로 1이 출력되고 c의 경우 null이 문자열처럼 취급되어 null is string이 출력되는 것을 확인할 수 있다.

String의 경우 asInstanceOf[String]이 붙어 처리되는 듯 하다.

String도 Option으로 처리해보면 어떨까?

{% highlight scala %}
case class A(a: Int, b: Long, c: String)

spark.sql("""select cast(null as int) as a, cast(null as long) as b, cast(null as string) as c""").as[A].map(r => A(Some(r.a.getOrElse(0) + 1), Some((r.b.getOrElse(0L) + 1L)), Some(r.c.getOrElse("") + " is string"))).toDF().show(1,false)
+---+---+----------+
|a  |b  |c         |
+---+---+----------+
|1  |1  | is string|
+---+---+----------+
{% endhighlight %}

위와 같이 String 또한 null일 경우 None으로 처리되는 것을 확인할 수 있다.

# 결론

따라서 Spark SQL 사용 시 null 값에 대한 확실한 처리를 위해서는

* UDF 작성 시 Primitive Type이 아닌 Object Type 사용
* Case Class 사용 시 Option 사용

을 유의해주어야 한다.