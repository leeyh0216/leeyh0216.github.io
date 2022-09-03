---
layout: post
title:  "Apache Spark 3.0에서 도입된 Adaptive Query Execution 알아보기"
date:   2022-09-04 00:20:00 +0900
author: leeyh0216
tags:
- apache-spark
---

> 이 글은 [Adaptive Query Execution: Speeding up Spark SQL at Runtime](https://www.databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)을 참고하여 작성하였습니다.

# Adaptive Query Execution

아마 마이크 타이슨에 대해 아는 사람이라면 아래의 명언을 들어본 적이 있을 것이다.

*Everyone has a plan, until they get punched in the mouth - 누구나 그럴싸한 계획을 가지고 있다. 주둥이를 처맞기 전까진, 마이크 타이슨*

실제로는 아래와 같은 말이지만...

*Everyone has a plan until they get hit. Then, like a rat, they stop in fear and freeze - 누구나 얻어맞기 전까진 계획을 가지고 있지만, 얻어맞으면 쥐처럼 공포에 떨고 얼어붙을 것이다*

왜 뜬금없이 마이크 타이슨의의 명언을 들고 왔는지 의아할 수 있지만, Adaptive Query Execution이야말로 이 명언에 가장 잘 어울리는 기능이라고 생각해서 소개해보았다.

[Spark Adaptive Query Execution](https://bomwo.cc/posts/sparkaqe/) 글에서도 원문을 기반으로 한글 번역본을 제공해주시는데, 이 글에서는 실제 겪었던 사례를 덧붙여서 내 관점으로 Adaptive Query Execution에 대한 분석을 해보려 한다.

## Adaptive Query Execution이 나오게 된 배경

모든 SQL 엔진은 사용자의 SQL을 그대로 실행하지 않고 최적화(Optimization) 과정을 거친 뒤 수행한다. 사용자의 SQL문이 최적화되지 않은 상태로 작성되었을 가능성이 크고, SQL문 내에서 사용되는 구문이나 테이블의 정보를 활용하면 더 좋은 방법으로 빠르게 구문을 실행할 수 있기 때문이다.

Apache Spark에서는 대략 다음의 과정을 거쳐 SQL문을 RDD로 바꾸어 쿼리를 수행한다.

1. Unresolved Logical Plan: SQL문을 단순 파싱하여 Abstract Syntax Tree(AST) 생성
2. Resolved Logical Plan: Unresolved Logical Plan에서 사용되는 테이블명, 컬럼명 등을 검증. 잘못된 테이블, 컬럼명을 사용하는 경우 이 단계에서 쿼리가 실패
3. Optimized Logical Plan: Resolved Logical Plan을 Catalyst Optimizer가 Rule을 적용하여 최적화된 Logical Plan 생성. Constant Folding, Predicate Pushdown, Projection Pruning, Null Propagation 등 다양한 기법들이 적용됨.
4. Physical Plan: 참조하는 테이블의 통계 정보(테이블 크기 등, Cost Based Optimizing)를 통해 실질적으로 클러스터에서 수행되는 Plan 생성(어떻게 파티셔닝하고 어떤 Join 전략을 사용할지 등). Physical Plan은 여러 개가 생성되며, 그 중 하나가 선택(Selected Physical Plan)되어 실질적으로 실행된다.
5. Codegen: Selected Physical Plan을 기반으로 Code Generation 수행. Tungsten이 개입

> 위 내용들은 [Understanding Spark's Logical and Physical Plan in layman's term](https://blog.knoldus.com/understanding-sparks-logical-and-physical-plan-in-laymans-term/)이나 [Deep Dive into Spark SQL's Catalyst Optimizer](https://www.databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html) 에서 더 세부적으로 다루고 있으니 참고 부탁드립니다.

이 중 Physical Plan이 문제가 되는데, Cost Based Optimizing 과정은 낮은 단계의 Stage에서만 효과를 발휘할 수 있다는 것이다. 예를 들어 아래와 같은 상황을 가정해보자.

* A와 B 테이블을 JOIN하여 C 테이블을 생성
* D와 E 테이블을 JOIN하여 F 테이블을 생성
* C와 F 테이블을 JOIN하여 G 테이블을 생성

A, B, D, E 테이블의 경우 테이블에 기록된 통계 정보를 기반으로 JOIN 등을 최적화 할 수 있지만, 이를 기반으로 Runtime에 생성되는 C와 F 테이블의 정보는 유추가 불가능하다. 그렇기 때문에 C와 F 테이블의 JOIN은 Optimizer가 최적화할 수 없으며 이 단계에서 개발자의 개입(이라 말하고 튜닝)이 들어가야 했다. 결과적으로 쳐맞기 전(C와 F 테이블을 JOIN하기 전)까지는 그럴듯한 계획을 세운 것 같았지만, C와 F 테이블이 JOIN되는 순간 최적화가 또 필요하다는 점이다.

Adaptive Query Execution은 쿼리 실행 전 한 번만 최적화를 진행하지 않고, 런타임에 점진적으로 최적화를 수행한다. 이 과정이 쿼리가 수행되는 상황에 적응해나간다는 의미에서 아마 Adaptive 라는 용어를 사용한 것 같다. 위의 상황을 가져와서 Adaptive Query Execution이 어떻게 동작하는지 정리해본다.

* A와 B 테이블을 JOIN하여 C 테이블을 생성 <- 이 과정에서 C에 대한 통계 정보가 생성되었음
* D와 E 테이블을 JOIN하여 F 테이블을 생성 <- 이 과정에서 F에 대한 통계 정보가 생성되었음
* C와 F 테이블의 통계 정보를 기반으로 JOIN 전략을 생성
* C와 F 테이블을 JOIN하여 G 테이블을 생성

이전에 없었던 세번째 단계가 추가되었고, 런타임에 한번 더 최적화를 수행하였다. 일반적으로 Stage가 병합되는 상황(Shuffle이 발생하는 시점에 Stage가 병합, 보통 JOIN)에 각 Stage의 통계 정보가 생성되고, 다음 Stage를 실행하기 전에는 이전 Stage가 모두 완료되어야 하기 때문에 자연스럽게 최적화 또한 가능해지는 것이다.

그럼 이제 Adaptive Query Execution에서 지원하는 세가지 최적화 기법에 대해서 상황에 맞추어 알아보기로 하자.

## Adaptive Query Execution의 최적화 기법

### Dynamically coalescing shuffle partitions

Spark SQL을 작성하며 가장 많이 튜닝하는 부분 중 하나가 JOIN 성능 튜닝일 것이다.

* JOIN Key의 편중(Skewness)이 없음. 아주 고르게 분포함
* JOIN하려는 두 테이블의 크기가 BHJ(Broadcast Hash Join)이 힘들 정도의 중/대규모 테이블
* 가용한 코어와 네트워크의 대역폭이 충분한 상태

위와 같은 상황에서 우리는 보통 `spark.sql.shuffle.partition` 값을 기본 값인 200보다 크게 설정하므로써 이슈를 해결해 왔을 것이다.

반대로 JOIN Key의 카디널리티가 낮은데, 필요 이상으로 `spark.sql.shuffle.partition` 값이 커서 일부 Task의 처리 대상 파티션의 데이터가 적은 경우 이 값을 작게 설정했을 수도 있다.

이러한 튜닝을 자동화 해주는 기능이 Adaptive Query Execution의 Dynamically coalescing shuffle partitions 이다. 이 기능은 `spark.sql.adaptive.coalescePartitions.enabled`를 `true`로 설정하면 활성화되며, `spark.sql.adaptive.advisoryPartitionSizeInBytes` 값을 기준으로 작은 파티션들을 Merge 하거나 큰 파티션들을 Split할 기준 크기를 제공할 수 있다.

### Dynamically switching join strategies

중/대규모 테이블과 소규모 테이블을 JOIN 할 때(보통 큰 쪽이 Fact Table, 작은 쪽이 Dimension 테이블), Broadcast Hash Join을 사용할 수 있다.

기존 Catalyst Optimizer도 소스 테이블이 작은 경우 Sort Merge Join이 아닌 Broadcast Hash Join으로 변경해주긴 했지만, 이미 한번 가공된 데이터들끼리 Join 하는 경우 우리가 임의로 한 쪽 테이들에 대해 `broadcast` 메서드를 호출하거나 Hint를 제공하는 방식으로 튜닝해왔다.

Adaptive Query Execution에서는 Dynamically switching join strategies 를 통해 이 문제를 해결하였고, 중간 테이블의 경우에도 `spark.sql.adaptive.autoBroadcastJoinThreshold` 보다 작은 테이블에 대해서는 BHJ 전략을 사용하는 것으로 보인다.

### Dynamically optimizing skew joins

튜닝하기 가장 까다로운 데이터 편중 상황에서의 JOIN 문제이다. 보통 중/대규모 테이블과 중규모 테이블의 JOIN에서 발생하는 문제이며, JOIN Key 중 일부 Key의 Row 수가 극단적으로 많을 때 발생한다.

보통 UI 상에서 다른 Task는 모두 끝났는데 1개의 Task만 오래 걸린다거나, SQL DAG 상에서 하나의 Exchange에서 데이터 크기가 편중된 것을 기반으로 해당 이슈가 발생했다는 것을 추정할 수 있다.

이 문제는 설정 튜닝만으로는 해결되지 않고, 코드로 해결해야 하는 문제이다. 나는 2015년에 외국 개발자가 소개한 [Fighting the skew in Spark](https://datarus.wordpress.com/2015/05/04/fighting-the-skew-in-spark/) 이라는 글을 기반으로 튜닝을 했었다. 해당 튜닝 방식이 그대로 Adaptive Query Execution에서 적용되었다.

Adaptive Query Execution을 사용하지 않고 해당 문제를 해결하는 방법을 우선 소개하겠다.

* A 테이블과 B 테이블을 JOIN
* A 테이블은 전체 건수가 100억건
* B 테이블은 전체 건수가 1000만건
* JOIN Key는 INT 값으로 1 ~ 100의 값을 가짐
* A 테이블의 JOIN Key 중 "1"에만 99억건의 데이터가 존재하는 상황
* B 테이블에는 JOIN Key가 편중되지 않고 적절히 분포함

위 경우 1의 값을 갖는 JOIN Key를 포함하는 Partition을 처리하는 Task가 끝나지 않고 계속 수행될 것이다. 이 문제를 해결하려면 99억건의 데이터를 N 개의 블록으로 쪼개되, 결과에는 영향을 미치지 않아야 한다.

1. A 테이블에 임의의 컬럼 `r`을 추가한다. `r`은 1 ~ 9의 값을 갖는 INT 형 컬럼이다.
2. B 테이블에 임의의 컬럼 `r_arr`을 추가한다. `r_arr`은 `[1, 2, 3, 4, 5, 6, 7, 8, 9]` INT 배열을 갖는 컬럼이다.
3. B 테이블의 `r_arr`에 `explode` 함수를 적용하여 `r` 컬럼을 만든다. 이 과정에서 B 테이블의 전체 Row 수는 1000만건에서 9000만건이 된다.
4. A 테이블과 B 테이블 JOIN 시 기존의 조인 조건(`A.k = B.k`라 가정)에 `A.r = B.r`을 AND 조건으로 추가한다.

```
val tblA = spark.read.parquet("location").withColumn("r", (rand() * 10) % 9 + 1)
val tblB = spark.read.parquet("location").withColumn("r_arr", lit(Array(1,2,3,4,5,6,7,8,9))).withColumn("r", explode("r_arr"))
tblA.join(tblB, tblA("k") <=> tblB("k") and tblA("r") <=> tblB("r"), "inner")
```

A 테이블의 문제가 되었던 99억개의 데이터는 11억개 정도씩 9개로 분할되어 병렬성을 9배 높일 수 있게 된다. 결과에 영향을 미치지 않기 위해 B 테이블의 어떠한 데이터라도 A 테이블과 JOIN 될 수 있도록 A 테이블의 랜덤 값을 모두 갖는 방식으로 뻥튀기 시켰다.

이러한 복잡한 과정을 Adaptive Query Execution에서는 `spark.sql.adaptive.skewJoin.skewedPartitionFactor`, `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` 만으로 해결하고 있다.

`spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` 이상의 크기를 가지는 파티션은 Skew 되었다고 판단하고, 이 데이터를 N개로 쪼개어 처리할 수 있도록 `spark.sql.adaptive.skewJoin.skewedPartitionFactor`을 제공하고 있다.

# 정리

개발자가 일일히 DAG를 보며 튜닝해야 했던 부분들을 이제는 프레임워크 수준에서 튜닝해준다. 한결 편해지긴 했지만 데이터 엔지니어의 역할이 점점 줄어들고 있음은 명확하고, 이제 설정 튜닝만으로도 대부분의 문제를 해결할 수 있지 않을까 싶다.

이제는 많은 케이스를 접하는 것이 훨씬 중요한 시기가 된 것 같다.