---
layout: post
title:  "Apache Druid - Getting started"
date:   2019-10-26 22:00:00 +0900
author: leeyh0216
tags:
- apache-druid
- study
---

> 스터디를 위해 Apache Druid 공식 문서를 번역/요약한 문서입니다.

# Getting started

## Introduction to Apache Druid

### Druid란?

Apache Druid는 대규모 데이터를 대상으로 **slice-and-dice** 분석을 지원하기 위해 만들어진 **실시간 분석용 데이터베이스**이다.

> slice and dice? 정보를 작은 단위로 쪼개어 다각도에서 바라볼 수 있도록 하는 것
>
> 출처: [What is slice and dice?](https://whatis.techtarget.com/definition/slice-and-dice)

Druid의 코어 아키텍쳐는 Data Warehouse, Time Series Database 그리고 Log Search Systems의 아이디어를 합쳐 만들었다. Druid의 특징은 다음과 같다.

* Columnar storage format: Druid는 컬럼 기반 스토리지를 사용하기 때문에, 쿼리 수행 시 필요한 컬럼만을 로드하므로 적은 컬럼을 조회할 경우엔 빠르게 동작한다. 또한 각 컬럼은 데이터 타입에 따라 빠르게 Scan/Aggregation 될 수 있도록 저장된다.

* Scalable distribution system: Druid는 수십 ~ 수백 대의 서버로 이루어진 클러스터로 구성되며, 초당 수백만 건의 수집 속도와 수십조 건의 저장 능력, 초 단위의 지연을 제공한다.

* Massively parallel processing: Druid는 클러스터 위에서 쿼리를 병렬로 수행할 수 있다.

* Realtime or batch ingestion: Druid는 실시간/배치로 데이터를 수집할 수 있다.

* Self-healing, self-balancing, easy to operate: Druid는 쉽게 클러스터에 노드를 추가/삭제할 수 있으며, 클러스터는 Downtime 없이 백그라운드에서 Rebalancing을 자동으로 수행한다. 또한 Druid 서버가 다운되는 경우 해당 서버가 대체될 때까지 시스템이 요청을 다른 서버로 라우팅한다. 마지막으로 Druid는 설정 변경, 소프트웨어 업데이트를 포함한 모든 경우에도 Downtime 없이 24/7 운영이 가능하도록 설계되었다.

* Cloud-native, falut-tolerant architecture that won't lose data: Druid는 데이터를 수집한 후 Deep storage(Cloud storage, HDFS, Shared filesystem 같은 저장소)에 데이터를 복사해 놓는다. 때문에 Druid 서버 하나가 죽더라도 데이터는 Deep storage로부터 복구될 수 있다. 몇몇 Druid 서버가 죽는 상황에도 복제 구성이 계속해서 쿼리를 수행할 수 있게 해준다.

* Indexes for quick filtering: Druid는 CONCISE 혹은 Roaring 압축 비트맵 인덱스 기능을 사용하여 인덱스를 생성한다. 이는 여러 개의 컬럼에 대한 빠른 필터링 속도를 제공한다.

* Time-based partitioning: Druid는 기본적으로 time 값을 기반으로 파티셔닝을 수행하고, 추가적으로 다른 필드들로도 파티셔닝이 가능하다. 이것은 time-based 쿼리가 해당 시간 대의 Segment에만 접근하고, 결론적으로 time-based 쿼리에 대한 성능 향상을 이끌어낸다.

* Approximate algorithm: Druid는 approximate count-distinct, approximate ranking 등의 기능을 제공한다. 

* Automatic summarization at ingestion time: Druid는 선택적으로 수집 단계에서의 데이터 Summarization 기능을 제공한다. 이는 부분적으로 데이터를 pre-aggretation하여 성능 향상을 제공한다.

### When should I use Druid?

* 데이터 생성 비율은 높지만, 업데이트 비율은 낮은 경우
* 대부분의 쿼리가 집계 연산이거나 리포트를 위한 쿼리인 경우
* 100밀리초 ~ 수초 이내의 지연시간으로 쿼리하고 싶은 경우
* 데이터가 시간 필드를 가지는 경우
* 카디널리티가 높은 데이터 컬럼에 대해 빠른 카운트/랭킹 쿼리를 수행해야 할 경우

### When should I not use Druid?

* Primary Key를 이용하여 이미 존재하는 데이터를 업데이트해야 하는 경우. Druid는 Streaming insert는 지원하지만 Streaming update는 지원하지 않는다.(다만 Background 배치 작업에서는 update 지원한다)
* 쿼리 지연 시간이 중요하지 않은 경우
* 큰 데이터의 Join이 필요한 경우

## Quick Start

> Quick Start의 경우 대부분 따라하면 되는 부분이 많기 때문에 별도로 정리하지는 않았음.
>
> 다만 진행 중 문제가 있었던 부분에 대해서만 정리하였음.

### 이미 설치된 Zookeeper가 가동되고 있을 경우

Druid 설치 이전에 이미 Zookeeper를 설치하여 가동하고 있었다.

```localhost:2181```로 가동되고 있었기 때문에, 해당 Zookeeper를 사용하겠지 하고 ```start-nano-quickstart``` 명령어를 수행할 경우 아래와 같이 오류가 발생한다.

```
seunyongui-MacBook-Pro:bin leeyh0216$ ./start-nano-quickstart
Cannot start up because port[2181] is already in use.
```

문제 해결을 하려고 ```${DRUID_HOME}/conf/supervise/single-server/nano-quickstart.conf``` 파일의 ```!p10 zk bin/run-zk conf``` 부분을 주석처리 했으나 위와 동일한 오류가 뜬다.

이 때는 가동 중인 Zookeeper를 종료하고, 설치된 Zookeeper를 ${DRUID_HOME}/zk 경로로 링크해준 다음 ```start-nano-quickstart``` 명령어를 수행하면 된다.