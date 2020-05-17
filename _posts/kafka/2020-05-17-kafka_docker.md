---
layout: post
title:  "Docker(compose)로 Kafka Cluster 실행하기"
date:   2020-05-17 22:10:00 +0900
author: leeyh0216
tags:
- kafka
- docker
---

# 개요

Kafka 공부 중 Replication 등을 테스트하기 위해 Kafka를 3대로 구성하여 Kafka Cluster를 띄워야 했다. 로컬 환경에 포트와 데이터 경로만 바꿔 실행할 수도 있었지만, 실행/종료가 귀찮을 것이 분명했기 때문에 Kafka를 Docker Image로 만든 뒤 Docker Compose를 통해 띄울 수 있도록 만들어 보았다.

처음에는 엄청 쉽게 될 것이라 생각했는데, 생각지도 못한 곳에서 삽질을 하게 되어 그 기록을 정리한다.

# Kafka Dockerfile 작성하기

## 기본 설정

Ubuntu 16.04 이미지를 베이스로 하였으며, Java의 경우 OpenJDK8 기반의 JRE를 사용하였다.

```
FROM ubuntu:16.04
  
LABEL Maintainer="leeyh0216@gmail.com"

# Change default mirror to kakao.
RUN sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

RUN apt-get update && apt-get install -y openjdk-8-jre wget

ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```

Ubuntu 기본 Repo가 느리기 때문에 kakao로 변경하였으며, Kafka Binary를 다운받기 위해 wget 또한 설치하였다.

## Kafka 설치

Kafka는 Naver의 Mirror에서 다운로드하였으며, /usr/local 경로에 설치하였다.

```
RUN wget http://mirror.navercorp.com/apache/kafka/2.5.0/kafka_2.12-2.5.0.tgz -O /usr/local/kafka.tgz && tar -xvf /usr/local/kafka.tgz -C /usr/local/ && rm /usr/local/kafka.tgz

ENV KAFKA_HOME=/usr/local/kafka_2.12-2.5.0
```

> wget의 -O 옵션을 사용하면 다운로드 받은 파일을 어떤 경로에 어떤 파일명으로 저장할 지 지정할 수 있다.

## `start-kafka.sh` 스크립트 작성

Docker Container 실행 시 전달된 환경 변수로 Kafka의 `server.properties`를 수정하고 Kafka Process를 실행하기 위한 `start-kafka.sh` 스크립트를 작성하였다.

`server.properties`에서 수정한 값들은 아래와 같다.

* `broker.id`: `${KAFKA_BROKER_ID}` 환경변수를 받아 지정할 수 있게 하였다. 해당 환경변수를 지정하지 않은 경우 -1(임의의 ID로 생성됨)을 사용하도록 했다.
* `zookeeper.connect`: `${ZOOKEEPER_SERVERS}` 환경변수를 받아 지정할 수 있게 하였다. 해당 환경변수를 지정하지 않은 경우 오류를 발생시키도록 하였다.

### 추가로 설정한 부분(삽질 포인트)

위의 `broker.id`와 `zookeeper.connect`만 설정한 뒤 Docker Container를 실행(포트는 9092:9092로 매핑)하였으나, 호스트 장비에서 `kafka-topics.sh` 스크립트를 통해 Topic을 생성하려고 시도했을 때 아래와 같은 오류를 마주하였다.

```
[2020-05-17 17:30:20,000] WARN Connection to node 0 could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```

이 문제를 이해하기 위해서는 `listeners`, `advertised.listeners` 속성과 클라이언트가 Kafka에 연결되는 과정에 대한 이해가 필요하다.

클라이언트가 가진 `bootstrap.servers`에는 Kafka Cluster에 포함된 모든 서버 목록이 존재하지 않을 수 있다. 때문에 클라이언트가 `bootstrap.servers`에 기재된 서버로 연결을 수행하면 연결된 서버는 Zookeeper에 등록된 Broker 서버들의 목록을 클라이언트에게 전달하고, 클라이언트는 이 정보로 Kafka Broker들에 연결할 수 있다.

`listeners`에 등록된 주소들은 Broker의 Server Socket을 Binding 할 주소이고, `advertised.listeners`는 Zookeeper에 등록하여 클라이언트가 해당 주소로 Broker에 접근할 수 있게 하는 속성이다.

정리하자면 나는 두 속성 모두 설정하지 않았기 때문에 Docker Container 내부 Host 명을 통해 접속을 시도하려 했고, 이 때문에 오류가 발생한 것이었다.

정상적으로 접근하게 하려면 아래와 같이 설정해주면 된다.

* `listeners`: `PLAINTEXT://{hostname}:9092,PLAINTEXT_HOST://0.0.0.0:29092`(Docker 네트워크에서의 호스트명을 가진 리스너 주소와 어떤 주소로든 접근 가능한 리스너 주소를 기입)
* `advertised.listeners`: `PLAINTEXT://{hostname}:9092,PLAINTEXT_HOST://{호스트 장비 IP}`(Docker 네트워크 내부에서 접근할 주소와 외부에서 접근할 주소를 기입)
* `listener.security.protocol.map`: `PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT`
* `inter.broker.listener.name`: `PLAINTEXT`

아래와 같이 `start-kafka.sh`에 작성하였다.

```
if [[ "${KAFKA_ADVERTISED_HOSTNAME}" == "" ]];
then
    echo "listeners=PLAINTEXT://`hostname`:9092" >> ${KAFKA_HOME}/config/server.properties
    echo "advertised.listeners=PLAINTEXT://`hostname`:9092" >> ${KAFKA_HOME}/config/server.properties
else
    if [[ "${KAFKA_ADVERTISED_PORT}" == "" ]];
    then
       KAFKA_ADVERTISED_PORT=29092
    fi
    echo "listener.security.protocol.map=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT" >> ${KAFKA_HOME}/config/server.properties
    echo "listeners=PLAINTEXT://`hostname`:9092,PLAINTEXT_HOST://:${KAFKA_ADVERTISED_PORT}" >> ${KAFKA_HOME}/config/server.properties
    echo "advertised.listeners=PLAINTEXT://`hostname`:9092,PLAINTEXT_HOST://${KAFKA_ADVERTISED_HOSTNAME}:${KAFKA_ADVERTISED_PORT}" >> ${KAFKA_HOME}/config/server.properties
    echo "inter.broker.listener.name=PLAINTEXT" >> ${KAFKA_HOME}/config/server.properties
fi
```

# 최종본

## Dockerfile

```
FROM ubuntu:16.04
  
LABEL Maintainer="leeyh0216@gmail.com"

# Change default mirror to kakao.
RUN sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

RUN apt-get update && apt-get install -y openjdk-8-jre wget

ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64 \
    KAFKA_HOME=/usr/local/kafka_2.12-2.5.0

RUN wget http://mirror.navercorp.com/apache/kafka/2.5.0/kafka_2.12-2.5.0.tgz -O /usr/local/kafka.tgz && tar -xvf /usr/local/kafka.tgz -C /usr/local/ && rm /usr/local/kafka.tgz

COPY start-kafka.sh ${KAFKA_HOME}/bin/start-kafka.sh

ENV ZOOKEEPER_SERVERS=localhost:2181

# Change user and start airflow process
RUN useradd -ms /bin/bash kafka && chown -R kafka:kafka ${KAFKA_HOME}
USER kafka

EXPOSE 29092

ENTRYPOINT ["/usr/local/kafka_2.12-2.5.0/bin/start-kafka.sh"]
```

## `start-kafka.sh`

```
#!/bin/bash
  
if [[ "${KAFKA_BROKER_ID}" == "" ]];
then
    KAFKA_BROKER_ID=-1
fi
sed -i "s/broker.id=0/broker.id=${KAFKA_BROKER_ID}/g" ${KAFKA_HOME}/config/server.properties

if [[ "${ZOOKEEPER_SERVERS}" == "" ]];
then
    echo "\${ZOOKEEPER_SERVERS} cannot be empty string"
    exit 1;
fi
sed -i "s/localhost:2181/${ZOOKEEPER_SERVERS}/g" ${KAFKA_HOME}/config/server.properties

if [[ "${KAFKA_ADVERTISED_HOSTNAME}" == "" ]];
then
    echo "listeners=PLAINTEXT://`hostname`:9092" >> ${KAFKA_HOME}/config/server.properties
    echo "advertised.listeners=PLAINTEXT://`hostname`:9092" >> ${KAFKA_HOME}/config/server.properties
else
    if [[ "${KAFKA_ADVERTISED_PORT}" == "" ]];
    then
       KAFKA_ADVERTISED_PORT=29092
    fi
    echo "listener.security.protocol.map=PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT" >> ${KAFKA_HOME}/config/server.properties
    echo "listeners=PLAINTEXT://`hostname`:9092,PLAINTEXT_HOST://:${KAFKA_ADVERTISED_PORT}" >> ${KAFKA_HOME}/config/server.properties
    echo "advertised.listeners=PLAINTEXT://`hostname`:9092,PLAINTEXT_HOST://${KAFKA_ADVERTISED_HOSTNAME}:${KAFKA_ADVERTISED_PORT}" >> ${KAFKA_HOME}/config/server.properties
    echo "inter.broker.listener.name=PLAINTEXT" >> ${KAFKA_HOME}/config/server.properties
fi

exec ${KAFKA_HOME}/bin/kafka-server-start.sh ${KAFKA_HOME}/config/server.properties
```

## 실행 명령어

```
> docker pull leeyh0216/kafka:0.1
> docker run --rm --hostname kafka --name kafka --env KAFKA_BROKER_ID=0 --env ZOOKEEPER_SERVERS={ZOOKEEPER서버 IP}:2181 --env KAFKA_ADVERTISED_HOSTNAME={호스트장비 IP} --env KAFKA_ADVERTISED_PORT=29092 kafka:0.1
```

# 참고한 자료

* [kafka trouble shooting: remote connect error](https://medium.com/@eun9882/kafka-trouble-shooting-remote-connect-error-a7970b00ffca)
* [Running Kafka in Docker Machine](https://medium.com/@marcelo.hossomi/running-kafka-in-docker-machine-64d1501d6f0b)