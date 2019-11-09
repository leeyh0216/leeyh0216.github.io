---
layout: post
title:  "Apache Spark - Detected implicit cartesian product for INNER join between logical plans"
date:   2019-11-09 23:00:00 +0900
author: leeyh0216
tags:
- apache-spark
---

> Spark에서의 Join 상황에서 발생할 수 있는 "Detected implicit cartesian product for INNER join between logical plans" 이슈에 대해서 정리하였습니다.

## 문제 상황

다음과 같은 두 개의 테이블이 존재한다고 가정하자.

#### 사용자(users)

| idx | name  | age |
|-----|-------|-----|
| 1   | user1 | 20  |
| 2   | user2 | 21  |
| 3   | user3 | 22  |

#### 친구(friends)

| idx1 | idx2 |
|------|------|
| 1    | 2    |
| 2    | 3    |

위 두 테이블을 사용하여 친구 관계인 사용자들의 이름, 나이를 아래와 같이 출력하고 싶다.

| name1 | name2 | age1 | age2 |
|-------|-------|------|------|
| user1 | user2 | 20   | 21   |
| user2 | user3 | 21   | 22   |

그럼 다음과 같이 SQL문을 작성할 수 있다.

{% highlight sql %}
SELECT
  A.name AS name1,
  B.name AS name2,
  A.age AS age1,
  B.age AS age2
FROM friends F
INNER JOIN users A ON A.idx = F.idx1
INNER JOIN users B ON B.idx = F.idx2
{% endhighlight %}

위 상황을 Spark에서 실행하면 "Detected implicit cartesian product for INNER join between logical plans"와 같은 오류를 접할 수 있는데, 이에 대해 확인해보고 회피할 수 있는 방안에 대해 고민해보았다.

## 테스트 데이터 준비

실제 상황과 유사하게 만들기 위해 데이터를 파일로부터 읽을 수 있도록 테스트 데이터를 생성/저장하였다.

{% highlight scala %}
import org.apache.spark.sql
import org.junit.{After, Before, Test}
import org.junit.rules.TemporaryFolder

class ReproduceCartesianProductTest {

  val spark = new sql.SparkSession.Builder().master("local[*]").config("spark.driver.host", "localhost").getOrCreate()

  val usersFolder = new TemporaryFolder()
  usersFolder.create()
  val usersPath = s"${usersFolder.getRoot.getAbsolutePath}/users"
  val friendsFolder = new TemporaryFolder()
  friendsFolder.create()
  val friendsPath = s"${friendsFolder.getRoot.getAbsolutePath}/friends"

  @Before
  def setUp(): Unit ={
    import spark.implicits._

    //사용자 테이블 생성
    Seq(
      (1, "User1", 20),
      (2, "User2", 24),
      (3, "User3", 25)
    ).toDF("idx", "name", "age").write.parquet(usersPath)

    //친구 테이블 생성
    Seq(
      (1, 2),
      (2, 3)
    ).toDF("idx1", "idx2").write.parquet(friendsPath)
  }

  @After
  def tearDown(): Unit ={
    usersFolder.getRoot.delete()
    friendsFolder.getRoot.delete()
  }
}

{% endhighlight %}

## Case 1. Spark View를 사용하여 구현하기

저장된 사용자(users)와 친구(friends) 테이블을 읽어 Temp View로 생성한 뒤, 위 SQL 구문을 실행해보았다.

{% highlight scala %}
  @Test
  def testWithTemporaryView(): Unit ={
    spark.read.parquet(usersPath).createOrReplaceTempView("users")
    spark.read.parquet(friendsPath).createOrReplaceTempView("friends")

    import spark.implicits._
    val expected = Seq(
      ("user1", "user2", 20, 21),
      ("user2", "user3", 21, 22)
    ).toDF("name1", "name2", "age1", "age2")

    val result = spark.sql(
      """
        |SELECT
        | A.name as name1,
        | B.name as name2,
        | A.age as age1,
        | B.age as age2
        |FROM friends F
        |INNER JOIN users A ON A.idx = F.idx1
        |INNER JOIN users B ON B.idx = F.idx2
        |""".stripMargin)

    result.explain(false)
    result.show(false)
    Assert.assertEquals(expected.count(), result.count())
    Assert.assertEquals(0, expected.except(result).count())
    Assert.assertEquals(0, result.except(expected).count())
  }
{% endhighlight %}

일단 아래와 같이 예상했던 결과가 출력되며,

|name1|name2|age1|age2|
|-----|-----|----|----|
|user2|user3|21  |22  |
|user1|user2|20  |21  |

Plan도 아래와 같이 사용자(users, A)와 친구(friends, B) 테이블이 먼저 Inner Join 된 후, 이 결과가 다시 사용자(users, B)와 Inner Join되는 것을 확인할 수 있다.

```
== Physical Plan ==
*(3) Project [name#28 AS name1#54, name#67 AS name2#55, age#29 AS age1#56, age#68 AS age2#57]
+- *(3) BroadcastHashJoin [idx2#34], [idx#66], Inner, BuildRight
   :- *(3) Project [idx2#34, name#28, age#29]
   :  +- *(3) BroadcastHashJoin [idx1#33], [idx#27], Inner, BuildLeft
   :     :- BroadcastExchange HashedRelationBroadcastMode(List(cast(input[0, int, true] as bigint)))
   :     :  +- *(1) Project [idx1#33, idx2#34]
   :     :     +- *(1) Filter (isnotnull(idx1#33) && isnotnull(idx2#34))
   :     :        +- *(1) FileScan parquet [idx1#33,idx2#34] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit2092067263039131031/..., PartitionFilters: [], PushedFilters: [IsNotNull(idx1), IsNotNull(idx2)], ReadSchema: struct<idx1:int,idx2:int>
   :     +- *(3) Project [idx#27, name#28, age#29]
   :        +- *(3) Filter isnotnull(idx#27)
   :           +- *(3) FileScan parquet [idx#27,name#28,age#29] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit9122563107872375344/..., PartitionFilters: [], PushedFilters: [IsNotNull(idx)], ReadSchema: struct<idx:int,name:string,age:int>
   +- BroadcastExchange HashedRelationBroadcastMode(List(cast(input[0, int, true] as bigint)))
      +- *(2) Project [idx#66, name#67, age#68]
         +- *(2) Filter isnotnull(idx#66)
            +- *(2) FileScan parquet [idx#66,name#67,age#68] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit9122563107872375344/..., PartitionFilters: [], PushedFilters: [IsNotNull(idx)], ReadSchema: struct<idx:int,name:string,age:int>

```

## Case 2. Spark Dataframe을 이용하여 구현하기

이 케이스가 문제가 발생하는 케이스이다. 다음과 같이 코드를 구현한 뒤, 실행해보자.

{% highlight scala %}
  @Test
  def testJoinWithDataframe(): Unit ={
    import org.apache.spark.sql.functions._

    val users = spark.read.parquet(usersPath)
    //테이블 구분을 위해 users Dataframe에 Transformation을 추가하여 새로운 Dataframe 생성
    val users1 = users.withColumn("tmp", lit(1))
    val users2 = users.withColumn("tmp", lit(2))
    val friends = spark.read.parquet(friendsPath)

    import spark.implicits._
    val expected = Seq(
      ("user1", "user2", 20, 21),
      ("user2", "user3", 21, 22)
    ).toDF("name1", "name2", "age1", "age2")

    val result = friends
      .join(users1, friends("idx1") <=> users1("idx"))
      .join(users2, friends("idx2") <=> users2("idx"))
      .select(
        users1("name").as("name1"),
        users2("name").as("name2"),
        users1("age").as("age1"),
        users2("age").as("age2")
      )

    result.explain(false)
    result.show(false)
    Assert.assertEquals(expected.count(), result.count())
    Assert.assertEquals(0, expected.except(result).count())
    Assert.assertEquals(0, result.except(expected).count())
  }
{% endhighlight %}

아래와 같은 Plan과 메시지만 출력되고 ```result.show(false)``` 구문은 실패한다.

```
== Physical Plan ==
org.apache.spark.sql.AnalysisException: Detected implicit cartesian product for INNER join between logical plans
Project [name#28, age#29]
+- Join Inner, ((idx1#43 <=> idx#27) && (idx2#44 <=> idx#27))
   :- Relation[idx1#43,idx2#44] parquet
   +- Relation[idx#27,name#28,age#29] parquet
and
Project
+- Relation[idx#82,name#83,age#84] parquet
Join condition is missing or trivial.
Either: use the CROSS JOIN syntax to allow cartesian products between these
relations, or: enable implicit cartesian products by setting the configuration
variable spark.sql.crossJoin.enabled=true;
```

오류 메시지:

```
Detected implicit cartesian product for INNER join between logical plans
Project [name#28, age#29]
+- Join Inner, ((idx1#43 <=> idx#27) && (idx2#44 <=> idx#27))
   :- Relation[idx1#43,idx2#44] parquet
   +- Relation[idx#27,name#28,age#29] parquet
and
Project
+- Relation[idx#82,name#83,age#84] parquet
Join condition is missing or trivial.
Either: use the CROSS JOIN syntax to allow cartesian products between these
relations, or: enable implicit cartesian products by setting the configuration
variable spark.sql.crossJoin.enabled=true;
org.apache.spark.sql.AnalysisException: Detected implicit cartesian product for INNER join between logical plans
Project [name#28, age#29]
+- Join Inner, ((idx1#43 <=> idx#27) && (idx2#44 <=> idx#27))
   :- Relation[idx1#43,idx2#44] parquet
   +- Relation[idx#27,name#28,age#29] parquet
and
Project
+- Relation[idx#82,name#83,age#84] parquet
Join condition is missing or trivial.
Either: use the CROSS JOIN syntax to allow cartesian products between these
relations, or: enable implicit cartesian products by setting the configuration
variable spark.sql.crossJoin.enabled=true;
	at org.apache.spark.sql.catalyst.optimizer.CheckCartesianProducts$$anonfun$apply$21.applyOrElse(Optimizer.scala:1129)
	at org.apache.spark.sql.catalyst.optimizer.CheckCartesianProducts$$anonfun$apply$21.applyOrElse(Optimizer.scala:1126)
	...
```

분명 원본 users 테이블에 Transformation을 적용하여 users1(A)와 users2(B)를 만들고 JOIN 했는데, Join Inner에서 사용된 Relation을 보면 users에 해당하는 테이블만 존재하고, Join 조건도 동일한 idx#27을 기준으로 idx1#43, idx2#44를 JOIN 하는 SELF JOIN 형태로 나온다.

위의 spark.sql.crossJoin.enabled=true로 놓고 JOIN해도 어쨋든 SELF JOIN 형태가 되어버리기 때문에 실행은 되지만 결과는 0건이 나오게 된다.

## 이러한 현상이 나타나는 이유는?

동일 데이터 소스로부터 만들어진 2개의 Dataframe은 아마도 내부적으로 구분되지 않는 듯 하다. [Why does spark think this is a cross cartesian join](https://stackoverflow.com/questions/42477068/why-does-spark-think-this-is-a-cross-cartesian-join) 이라는 글을 보아도 Join 과정에서 동일 Lineage를 참고하기 때문에 Join 조건이 Trivial 하기 때문이라고 나와 있다.

## 해결책

users1과 users2를 만들 때, **동일 Dataframe으로부터 만들어내지 않고 아예 각각 소스 데이터를 읽도록 하여 별개의 Lineage를 가진 Dataframe을 만드는 방식으로 우회**하면 된다.

{% highlight scala %}
  @Test
  def testJoinWithDataframeSuccessfully(): Unit ={
    //동일 소스라도 다른 Lineage를 갖도록 따로 읽어들인다.
    val users1 = spark.read.parquet(usersPath)
    val users2 = spark.read.parquet(usersPath)
    val friends = spark.read.parquet(friendsPath)

    import spark.implicits._
    val expected = Seq(
      ("user1", "user2", 20, 21),
      ("user2", "user3", 21, 22)
    ).toDF("name1", "name2", "age1", "age2")

    val result = friends
      .join(users1, friends("idx1") <=> users1("idx"))
      .join(users2, friends("idx2") <=> users2("idx"))
      .select(
        users1("name").as("name1"),
        users2("name").as("name2"),
        users1("age").as("age1"),
        users2("age").as("age2")
      )

    result.explain(false)
    result.show(false)
    Assert.assertEquals(expected.count(), result.count())
    Assert.assertEquals(0, expected.except(result).count())
    Assert.assertEquals(0, result.except(expected).count())
  }
{% endhighlight %}

위와 같이 실행할 경우 아래와 같이 Plan이 출력되는데,

```
== Physical Plan ==
*(3) Project [name#28 AS name1#99, name#34 AS name2#100, age#29 AS age1#101, age#35 AS age2#102]
+- *(3) BroadcastHashJoin [coalesce(idx2#40, 0)], [coalesce(idx#33, 0)], Inner, BuildRight, (idx2#40 <=> idx#33)
   :- *(3) Project [idx2#40, name#28, age#29]
   :  +- *(3) BroadcastHashJoin [coalesce(idx1#39, 0)], [coalesce(idx#27, 0)], Inner, BuildLeft, (idx1#39 <=> idx#27)
   :     :- BroadcastExchange HashedRelationBroadcastMode(List(cast(coalesce(input[0, int, true], 0) as bigint)))
   :     :  +- *(1) FileScan parquet [idx1#39,idx2#40] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit4528740988515346186/..., PartitionFilters: [], PushedFilters: [], ReadSchema: struct<idx1:int,idx2:int>
   :     +- *(3) FileScan parquet [idx#27,name#28,age#29] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit7951897441267757241/..., PartitionFilters: [], PushedFilters: [], ReadSchema: struct<idx:int,name:string,age:int>
   +- BroadcastExchange HashedRelationBroadcastMode(List(cast(coalesce(input[0, int, true], 0) as bigint)))
      +- *(2) FileScan parquet [idx#33,name#34,age#35] Batched: true, Format: Parquet, Location: InMemoryFileIndex[file:/var/folders/kf/6czlz6352pgdxyb95wtlln6c0000gn/T/junit7951897441267757241/..., PartitionFilters: [], PushedFilters: [], ReadSchema: struct<idx:int,name:string,age:int>

```

Temp View를 이용하여 만든 테스트와 동일한 Plan이 출력되는 것을 볼 수 있으며, Case 2의 상황과는 달리 users에 대한 Relation이 각각 생성되며, idx1#39와는 idx#27, idx2#40과는 idx#33과 조건을 비교하는 것을 확인할 수 있다.