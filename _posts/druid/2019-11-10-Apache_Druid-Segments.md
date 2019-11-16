---
layout: post
title:  "Apache Druid - Segments"
date:   2019-11-10 15:00:00 +0900
author: leeyh0216
tags:
- apache-druid
- study
---

> 스터디를 위해 Apache Druid 공식 문서를 번역/요약한 문서입니다.

# Segments

Apache Druid는 시간 기준으로 파티셔닝된 색인을 Segment 파일에 저장한다. 하나의 Segment 파일은 Ingestion 단계에서 정의한 granularitySpec의 segmentGranulairty 만큼의 Time Interval을 기준으로 생성된다.

Druid가 Heavy Query를 효율적으로 수행하게 하기 위해서는 Segment 파일의 크기가 300mb ~ 700mb 정도로 생성되도록 구성하는 것을 추천한다. ㄷ