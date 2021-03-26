---
layout: post
title:  "LEAD, LAG 함수"
date:   2021-03-26 23:00:00 +0900
author: leeyh0216
tags:
- sql
---

# 개요

데이터 분석 중 사용자의 로그들을 묶어서 봐야 하는 경우가 생겼다. 예를 들자면 아래와 같다.

**로그 테이블**

사용자(user)와 방문일(visit_date)로 구성된 테이블이다.

| user | visit_date |
|------|------------|
| A    | 2021-03-01 |
| A    | 2021-03-04 |
| A    | 2021-03-08 |
| B    | 2021-03-24 |

**내가 만들고 싶은 테이블**

사용자(user)와 방문일(visit_date), 그리고 다음 방문일(next_visit_date)로 구성된 테이블이다.

| user | dt         | next_visit |
|------|------------|------------|
| A    | 2021-03-01 | 2021-03-04 |
| A    | 2021-03-04 | 2021-03-08 |
| A    | 2021-03-08 | NULL       |
| B    | 2021-03-24 | NULL       |

## 나의 접근(바보같은 접근)

우선 Window Function인 `RANK()`를 통해 사용자(user)별로 로그의 순서를 정의한다.

| user | visit_date | ord |
|------|------------|-------|
| A    | 2021-03-01 | 1     |
| A    | 2021-03-04 | 2     |
| A    | 2021-03-08 | 3     |
| B    | 2021-03-24 | 1     |

위와 테이블 2개(A, B)를 LEFT OUTER JOIN하는데 JOIN 조건은 (A.user = B.user AND A.ord + 1 = B.ord)이다.

| user | visit_date | ord | user | visit_date | ord |
|------|------------|-------|------|------------|-------|
| A    | 2021-03-01 | 1     | A    | 2021-03-04 | 2     |
| A    | 2021-03-04 | 2     | A    | 2021-03-08 | 3     |
| A    | 2021-03-08 | 3     | NULL | NULL       | NULL  |
| B    | 2021-03-24 | 1     | NULL | NULL       | NULL  |

필요한 컬럼(A.user, A.visit_date, B.visit_date)만 가지고 결과 테이블을 만들어낸다.

위의 과정을 쿼리로 정리하면 다음과 같다.

```
SELECT
    A.user,
    A.visit_date AS visit_date,
    B.visit_date AS next_visit_date
FROM
    (
        SELECT
            row_number() OVER (PARTITION BY user ORDER BY visit_date ASC) as ord,
            user,
            visit_date
        FROM
            user_visit
    ) A
LEFT OUTER JOIN
    (
        SELECT
            row_number() OVER (PARTITION BY user ORDER BY visit_date ASC) as ord,
            user,
            visit_date
        FROM
            user_visit
    ) B
ON 
    A.user = B.user AND A.ord + 1 = B.ord
ORDER BY
    A.user ASC, A.visit_date ASC
```

위 쿼리는 분산 환경에서 실행할 때 매우 비효율적이다.

테이블 2개를 만들어낼 때 `row_number`에서 이미 2번의 Shuffle이 발생하는데, JOIN에 의한 Shuffle까지 총 3번의 Shuffle이 발생하기 때문이다.

## 좋은 접근(LEAD 함수 활용하기)

LEAD 함수는 Window 함수 중 하나로써 다음 행의 특정 컬럼 값을 반환하는 기능을 제공한다. 우리는 user 별로 visit_date 정렬 후, 다음 visit_date를 가져와야 하므로 `LEAD(visit_date) OVER (PARTITION BY user ORDER BY visit_date ASC)` 같이 표현할 수 있다. 최종적으로 쿼리는 아래와 같이 간단해진다.

```
SELECT
    user,
    visit_date,
    LEAD(visit_date) OVER (PARTITION BY user ORDER BY visit_date ASC) AS next_visit_date
FROM
    user_visit;
```

성능 관점에서도 1번의 Shuffle만 발생할 것이기 때문에 JOIN 방식보다 훨씬 좋다.

추가로 LAG는 LEAD와 다르게 이전 행의 특정 컬럼 값을 반환하는 기능을 제공한다. 위와 반대로 다음 방문 날짜가 아닌 이전 방문 날짜를 얻어내는 경우 LAG을 사용할 수 있겠다.