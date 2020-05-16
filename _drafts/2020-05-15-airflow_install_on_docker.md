---
layout: post
title:  "5분만에 Apache-airflow Docker에 설치하고 실행해보기"
date:   2029-05-16 00:30:00 +0900
author: leeyh0216
tags:
- docker
- airflow
---

# 개요

2017년 경에 설치 후 튜토리얼을 실행해본 적은 있었는데, 앞으로 사용할 일이 많아질 것 같아 미리 설치부터 테스트까지 진행해보려 한다.

집에서 사용하는 노트북은 맥북이지만, 운영 환경은 아무래도 우분투가 설치되어 있을 가능성이 크기 때문에 도커를 통해 아래와 같은 환경에서 설치/실행해보려 한다.

## 설치 환경 및 버전

* Docker 19.03.5
* Ubuntu 16.04 LTS
* Python 3.7
* Airflow 1.10.10

# 설치해보기

개인적으로 파이썬에 익숙하지 않고 Dockerfile도 한번에 써내려갈 자신이 없었기 때문에, Ubuntu 16.04 이미지를 컨테이너로 띄워서 삽질을 하며 진행했다.

## Docker Container 생성

테스트로 설치를 수행할 Docker Container를 생성한다. Ubuntu 16.04 이미지를 기반으로 진행할 것이다. 아래 명령어는 Docker가 설치된 Host 머신에서 수행한 명령어이다.

```
> docker run -it --name airflow -p 8081:8081 ubuntu:16.04
```

컨테이너 안에서 Airflow 웹서버 실행 후 확인할 것이기 때문에 Host 머신의 8081 포트를 Container의 8081 포트로 매핑해주었다.

## `sources.list` 업데이트

> 굳이 수행하지 않아도 되는 *Optional*한 과정입니다.

공식 Ubuntu 이미지를 받으면 `sources.list` 파일의 Mirror가 기본 Mirror(archive.ubuntu.com)으로 설정되어 있는데, 해외 서버라서 엄청 느리기 때문에 국내 Mirror로 변경하는 과정을 거친다. 나는 Kakao의 Mirror로 변경하였다.

```
> sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list
```

## 패키지 목록 업데이트 및 파이썬 설치

패키지 목록 업데이트와 파이썬 설치를 진행한다.

### 패키지 목록 업데이트

아래 명령어를 통해 패키지 업데이트를 수행한다.

```
> apt-get update
```

### 파이썬 설치

파이썬을 설치하기 전에 `software-properties-common` 패키지를 우선하여 설치한다.

```
> apt-get install software-properties-common
```

또한 파이썬 3.7을 제공하는 패키지 저장소를 추가해야 한다. 구글을 뒤져보니 `deadsnakes`라는 팀의 저장소를 많이 사용하는 것 같아 해당 저장소로 추가하였다.

```
> add-apt-repository -y ppa:deadsnakes/ppa
```

이제 파이썬 3.7 버전을 설치한다.

```
apt-get install -y python3.7 python3.7-dev
```

**주의해야 할 점은 `python3.7` 뿐만 아니라 `python3.7-dev`도 설치해야한다는 것이다.** `python3.7-dev`를 설치하지 않으면 나중에 pip3를 통해 apache-airflow 설치 중 아래와 같은 오류 메시지를 마주할 수 있다.

```
psutil/_psutil_common.c:9:20: fatal error: Python.h: No such file or directory
compilation terminated.
error: command 'x86_64-linux-gnu-gcc' failed with exit status 1
```

이후 `update-alternatives` 명령어를 통해 `/usr/bin/python3` 경로에 우리가 설치한 `python3.7` 버전의 심볼릭 링크를 걸어준다.

```
> update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 0
```

마지막으로 `pip3`를 설치하고 업그레이드한다.

```
> apt-get install -y python3-pip && pip3 install --upgrade pip
```

## Airflow 설치, 초기화 및 실행

### Airflow 설치

`pip`을 이용하여 Airflow를 설치한다.

```
> pip3 install apache-airflow
```

주의해야할 점은 **sudo 권한으로 설치를 진행해야한다는 점**이다. sudo 권한으로 설치를 진행하지 않을 경우 설치한 계정의 `~/.local/bin`에 Airflow가 설치되고 PATH에 해당 경로가 추가되지 않기 때문에 `airflow` 명령어 실행 시 `Command Not Found`라는 에러에 직면할 수 있다. 도대체 공식문서에서는 왜 이부분을 언급해주지 않는건지 모르겠다.(한참 삽질함..)

> 참고자료: [StackOverflow - Getting bash: airflow: command not found](https://stackoverflow.com/questions/51122849/getting-bash-airflow-command-not-found)

### Airflow Metadata Database 초기화

운영 환경에서는 MySQL, MariaDB와 같은 DB를 사용하겠지만, 테스트 환경에서는 Sqlite로도 충분할 것이다.

일단 Airflow의 설정 파일을 보관할 디렉토리를 만들고 해당 경로를 `AIRFLOW_HOME`이라는 환경변수로 EXPORT 한다.

```
> adduser airflow #airflow 사용자 계정 생성
> mkdir /home/airflow/airflow #airflow 설정 파일 디렉토리 생성
> export AIRFLOW_HOME=/home/airflow/airflow
```

이후 아래 명령어를 통해 Metadata database를 초기화한다.

```
> airflow initdb
```

### 실행

Airflow WebServer와 Scheduler를 실행해야한다. 아래 두 개의 명령어를 입력한다.

```
> airflow webserver -p 8081:8081 -D
> airflow schedule -D
```

정상적으로 실행되었다면 http://localhost:8081로 접속했을 떄 Airflow 화면이 나와야 한다.

# Dockerfile 작성하기

간단한 테스트 용도로 만든 Dockerfile이라 여러모로 부족하지만, 일단 Airflow를 띄우는 것이 목표였기 때문에  아래와 같이 작성하였다.

**Dockerfile**

```
FROM ubuntu:16.04
  
LABEL Maintainer="leeyh0216@gmail.com"

# Change default mirror to kakao.
RUN sed -i 's/archive.ubuntu.com/mirror.kakao.com/g' /etc/apt/sources.list

# Update mirrors and install packages
RUN apt-get update && apt-get install -y software-properties-common && add-apt-repository -y ppa:deadsnakes/ppa && apt-get update && apt-get install -y python3.7 python3.7-dev && update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.7 0 && apt-get install -y python3-pip && pip3 install --upgrade pip && pip3 install apache-airflow

# Change user and start airflow process
RUN useradd -ms /bin/bash airflow
USER airflow
WORKDIR /home/airflow

ENV AIRFLOW_HOME=/home/airflow/airflow
ENV AIRFLOW_LOG_DIR=${AIRFLOW_HOME}/logs
ENV AIRFLOW_SBIN_DIR=${AIRFLOW_HOME}/sbin

RUN mkdir ${AIRFLOW_HOME} && mkdir -p ${AIRFLOW_LOG_DIR} && mkdir -p ${AIRFLOW_SBIN_DIR}

COPY start-airflow.sh ${AIRFLOW_SBIN_DIR}

RUN airflow initdb

EXPOSE 8080

ENTRYPOINT ["/home/airflow/airflow/sbin/start-airflow.sh"]
```

**start-server.sh**

```
#!/bin/bash
  
AIRFLOW=/usr/local/bin/airflow

${AIRFLOW} scheduler -l ${AIRFLOW_LOG_DIR}/scheduler.log &

${AIRFLOW} webserver -p 8080 -A ${AIRFLOW_LOG_DIR}/airflow-access.log -E ${AIRFLOW_LOG_DIR}/airflow-error.log -l ${AIRFLOW_LOG_DIR}/airflow.log --stdout ${AIRFLOW_LOG_DIR}/airflow.log --stderr ${AIRFLOW_LOG_DIR}/airflow-error.log
```