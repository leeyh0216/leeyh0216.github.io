---
layout: post
title:  "[Hadoop] Yarn Overview"
date:   2017-03-11 22:07:00 +0900
author: leeyh0216
categories: dev hadoop yarn
---

> 이 문서는 [Apache Hadoop Yarn](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)을 번역한 문서입니다.

### Apache Hadoop Yarn

Yarn의 기본적인 아이디어는 리소스 관리와 잡 스케쥴링/모니터링을 다른 데몬으로 분리한 것입니다. Yarn은 Global ResourceManager(RM)와 Application Master(AM)로 나뉘어져 있습니다. Yarn에서 작동하는 Application은 Single Job 또는 Job의 Directed Acyclic Graph로 취급됩니다.

ResourceManager와 NodeManager는 Data Computation Framework를 구성합니다. Resource Manager은 Yarn을 구성하는 시스템의 전체 리소스 관리를 담당하니다. Node Manager는 Yarn을 이루는 각 서버의 Container를 관리하고 Container들의 Resource(ex. Cpu, Memory, Disk, Network) 관리 정보를 모니터링하고 ResourceManager에게 전송하는 역할을 담당합니다.

각 Application의 Application Master는 Resource Manager에게 자원을 할당받고, Node Manager에 의해 작업을 할당받아 수행하고 모니터링 됩니다.

Resource Manager는 2개의 주요 컴포넌트(Scheduler, ApplicationsMater)로 이루어져 있습니다.

Scheduler는 다양한 Application에게 자원을 할당합니다. 단순히 자원 할당의 역할만을 담당하고 있으므로 작업의 상태나 실패 복구 등에는 관여하지 않습니다.

ApplicationsMaster는 클라이언트가 요청한 작업을 제출받아 Application을 실행시키는 특별한 Container인 Application Master를 결정하고, 작업이 실패한 Container를 재실행 하는 등의 작업을 수행합니다.
