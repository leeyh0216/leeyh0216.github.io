---
layout: post
title:  "데이터 엔지니어를 위한 CMU Intro to Database Systems#1"
date:   2023-12-21 18:00:00 +0900
author: leeyh0216
tags:
- database
---

# 들어가며

2014년 2학기에 데이터베이스 개론을 수강했던 기억이 있다. 당시에는 개념적인 것보다 실제 동작하는 프로그램에 RDBMS를 연동하여 결과를 얻어내는 것에 더 관심이 있었다. 그러나 몇년 간 데이터 엔지니어 업무를 수행하다보니, 최적화를 위해서는 엔진 내부에 대한 이해가 중요하다는 결론이 내려졌다.

내가 다루는 엔진이 대부분 분산 데이터베이스 엔진이기는 하지만, 기본적인 개념은 일반 데이터베이스를 구성하는 이론과 크게 차이가 없는 것을 확인하였다. 어떤 자료를 통해 공부할지 고민하다가 카네기 멜론 대학교의 "CMU Intro to Database Systems"와 "CMU Advanced Database Systems" 강의를 듣기로 마음먹었다.

시간이 좀 많으면 모든 Lecture를 다 듣고 싶은데, 연말 휴가기간 동안 듣는 거라 필요하다고 생각되는 부분만 쏙쏙 빼서 공부하고 정리하기로 결정했다.

# Lecture #01: Relational Model & Algebra

참고자료: [Youtube - F2023 #01 - Relational Model & Algebra \(CMU Intro to Database Systems\)](https://youtu.be/XGMoq-D_mao?si=vHSbqxJe410WocsZ)

## Database Management System

Database Management System(이하 DBMS)는 데이터베이스에 데이터를 저장하거나, 데이터베이스에 존재하는 데이터를 활용할 수 있게 하는 소프트웨어를 의미한다. 범용적인 DBMS는 특정 데이터 모델을 기반으로 DML, DDL, Management 등을 수행할 수 있도록 설계된다.

**데이터 모델은 데이터베이스에 저장된 데이터를 표현하는 방식**을 의미한다. Relational Model, Key-Value Model, Vector Model 등 다양한 데이터 모델이 존재한다.

### Early DBMSs

초창기 DBMS들은 Logical Layer와 Physical Layer가 커플링되어 있어 유지보수가 힘들었다.

* Logical Layer: 데이터베이스의 개체와 속성을 표현하는 계층
* Physical Layer: 데이터베이스의 개체와 속성을 저장하는 계층

Physical Layer가 DBMS 코드에 직접적으로 노출되었기 때문에, Physical Layer가 변경되면 DBMS의 모든 코드를 변경해주어야 하는 상황이 벌어졌다.

## Relational Model

Physical Layer가 바뀔 때마다 DBMS 코드를 수정해야 하는 문제를 해결하기 위해, 1969년 Relation Model이 등장했다. Relational Model은 **Relation에 기반한 데이터베이스 추상화를 제공**한다. Relation Model은 다음 세가지 핵심 개념을 포함한다.

* 데이터베이스에 저장되는 간단한 **자료구조(Relation)**
* **High Level Language로 데이터에 접근**할 수 있어야 하며, DBMS는 이를 기반으로 **최적의 실행 계획**을 세울 수 있어야 함
* **데이터의 저장 방식은 DBMS의 구현**에 달려 있음

Relation Model 내의 개념들은 다음과 같다.

* `relation`: 개체(`entity`)를 표현하는 속성(`attribute`)들의 순서 없는 집합이다.
  * n개의 `attribute`로 된 `relation`을 n-ary `relation`이라고 한다.
  * 이제부터 `relation`은 테이블(`table`)로 표현할 것이며, n-ary `relation`은 n개의 컬럼을 가진 테이블을 의미한다.
* `tuple`은 `attribute` 값(`domain`)들의 집합이다.
  * `domain`은 `attribute`가 가질 수 있는 **원자 값들의 집합**을 의미한다.

## Relational Algebra

Relational Algebra는 Relation에 대해 Tuple을 생성하거나 조회하는 기본적인 Operator들을 의미한다. 각 Operator들은 1개 혹은 그 이상의 `relation`을 입력으로 받고, 새로운 Relation을 반환한다. Operator들을 결합(Chain)하므로써 쿼리를 만들어낼 수 있다.

### Select

Select는 Relation을 입력받아 조건(Predicate)에 맞는 Relation을 반환하는 Operator이다. Predicate는 Filter와 같이 동작하며, 여러 개의 Predicate를 논리곱(conjunction)과 논리합(conjunction)으로 결합하여 사용할 수 있다.

* Syntax: `σpredicate(R)`
* Example: `σa_id='a2'(R)`
* SQL: `SELECT * FROM R WHERE a_id = 'a2'`

### Projection

Projection은 Relation을 입력받아 주어진 Attribute만 포함하는 Tuple들로 구성된 Relation을 반환하는 Operator이다. 명시한 Attribute 순서로 구성된 Relation을 출력으로 가지게 된다.

* Syntax: `πA1,A2,...,An(R)`
* Example: `πb_id-100, a_id(σa_id=’a2’(R))`
* SQL: `SELECT b_id - 100, a_id FROM R WHERE a_id = 'a2'`

### Union

Union은 두 개의 Relation을 입력받아 두 Relation의 모든 Tuple을 포함한 Relation을 반환한다. **두 입력 Relation은 완전히 동일한 Attribute로 구성되어 있어야 한다**

* Syntax: `(R ∪ S)`
* SQL: `(SELECT * FROM R) UNION ALL (SELECT * FROM S)`

### Intersection

Intersection은 두 개의 Relation을 입력받아 두 개 Relation 모두에 존재하는 Tuple로 구성된 Relation을 반환한다. **두 입력 Relation은 완전히 동일한 Attribute로 구성되어 있어야 한다**

* Syntax: `(R ∩ S)`
* SQL: `(SELECT * FROM R) INTERSECT (SELECT * FROM S)`

### Difference

Difference는 두 개의 Relation을 입력받아 첫번째 Relation에는 포함되어 있지만, 두번째 Relation에는 포함되어 있지 않은 Tuple로 구성된 Relation을 반환한다. **두 입력 Relation은 완전히 동일한 Attribute로 구성되어 있어야 한다**

* Syntax: `(R − S)`
* SQL: `(SELECT * FROM R) EXCEPT (SELECT * FROM S)`

### Product

Product는 두 개의 Relation을 입력받아 입력 Relation들의 모든 조합으로 구성된 Relation을 반환한다.

* Syntax: `(R × S)`
* SQL: `(SELECT * FROM R) CROSS JOIN (SELECT * FROM S)`
  * 혹은 `SELECT * FROM R, S`

### Join

Join은 두 개의 Relation을 입력받아 두 Relation의 조합으로 구성된 Relation을 반환한다. 단, Product와는 다르게 두 Relation이 공유하는 각 Attribute의 값이 같은 Tuple 조합만이 반환된다.

* Syntax: `(R ▷◁ S)`
* SQL: `SELECT * FROM R JOIN S USING (ATTRIBUTE1, ATTRIBUTE2...)`

## Observation

Relational Algebra는 쿼리를 어떻게 처리하는지에 대한 High Level Language이다.

`σb_id=102(R ▷◁ S)` 표현은 R과 S Relation을 Join 한 뒤, `b_id`가 102인 Tuple로 구성된 Relation을 반환한다. 그리고 `(R▷◁(σb_id=102(S)))` 표현식은 S Relation에서 `b_id`가 102인 Tuple로 구성된 Relation을 추출한 뒤, R과의 Join을 수행하여 결과 Relation을 반환한다. 두 표현식은 완전히 같은 동작을 하는데, 만일 S Relation이 10억개의 Tuple로 구성되어 있고, `b_id`가 102인 Tuple은 하나 밖에 없다면, `(R▷◁(σb_id=102(S)))`의 속도가 훨씬 빠르게 동작할 것이다.

직접적으로 Relational Algebra를 RDBMS에 전달하는 것보다 어떤 결과를 얻고 싶다는 표현을 RDBMS에 전달하면, RDBMS가 해당 표현을 최적의 Relational Algebra로 변환하는 것이 더 나을 것이다. SQL이 이러한 표현을 의미하며, RDBMS는 SQL을 받아 최적의 Relational Algebra로 변환하는 역할을 수행하게 된다.