---
layout: post
title:  "[Hadoop] Installation"
date:   2017-03-02 00:40:00 +0900
author: leeyh0216
categories: hadoop dev
---

> 이 문서는 Hadoop 2.7.3 버전을 Ubuntu 14.04 Docker Image 에 설치하기 위해 작성되었습니다. Docker라는 특수한 환경 설정을 제외하면(처음 부분을 제외하면) 일반 Hadoop 2.7.3 버전 설치와 동일하게 진행할 수 있습니다

### Docker Container Setting

완전 분산 모드로 하둡을 설치하기 위해서 Name Node 1대, Data Node 2대를 위한 Docker Container를 생성해야 한다.

Docker Ubuntu 14.04 이미지를 세팅해주자

{% highlight bash %}
$ apt-get update
$ apt-get upgrade
$ apt-get install vim
$ apt-get install wget
{% endhighlight %}

Hadoop2는 내부적으로 Google Protobuf(Version 2.5)를 사용한다.
[Google Protobuf v2.5](https://github.com/google/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz) 를 클릭하면 다운받을 수 있지만, terminal 환경에서 다운받아야 하므로, 다음과 같은 명령어를 실행하자.


{% highlight bash %}
$ wget https://github.com/google/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz
$ tar -zxvf protobuf-2.5.0.tar.gz
$ cd protobuf-2.5.0.tar.gz
$ ./configure
$ make
$ make install
$ protoc --version
{% endhighlight %}


### Hadoop Installation

#### Hadoop2 다운로드

하둡2(2.7.2) 설치를 진행한다.
다운로드 주소는 다음과 같다 : http://mirror.navercorp.com/apache/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz

{% highlight bash %}
$ wget http://mirror.navercorp.com/apache/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz
$ tar -zxvf hadoop.2.7.3.tar.gz
$ cd hadoop.2.7.3.tar.gz
{% endhighlight %}

#### Hadoop 환경 설정 파일

Hadoop의 설정파일은 모두 {하둡 설치 경로}/etc/hadoop에 존재한다.
각 파일들은 다음과 같은 정보들을 가지고 있다.

- hadoop-env.sh : 하둡을 실행하는 쉘 스크립트 파일에서 필요한 환경 변수를 설정. JDK, classpath, Daemon 실행 옵션 등을 설정할 수 있다.
- masters : 보조 네임노드를 실행할 서버 설정
- slaves : 데이터 노드를 실행할 서버 설정
- core-site.xml : HDFS와 맵리듀스에서 공통적으로 사용할 환경 정보를 사용한다.
- hdfs-site.xml : HDFS에서 사용할 환경 정보를 설정한다.
- mapred-site.xml : 맵리듀스에서 사용할 환경 정보를 설정한다.
- yarn-env.sh : 얀을 실행하는 쉘 스크립트 파일에서 필요한 환경변수를 설정한다.
- yarn-site.xml : 얀에서 사용할 환경 정보를 설정한다.

#### hadoop-env.sh 수정

hadoop-env.sh 파일에서는 Hadoop이 사용하는 JAVA JDK Directory와 HADOOP_PID_DIR만 설정해주면 된다.

파일 내용 중 export JAVA_HOME=${JAVA_HOME}으로 되어 있는 부분을 컴퓨터에 설치된 JAVA Directory로 설정해주자(which java)
또한 Hadoop Daemon의 PID 저장 경로인 HADOOP_PID_DIR을 적당히 설정해준다.

#### slaves 수정

slaves 파일에는 데이터노드 호스트 목록을 설정한다.
데이터 노드 호스트 등록 시에는 반드시 /etc/hosts에 등록된 이름과 일치하는지 확인 후에 기재한다.
예를 들어, hadoop02,hadoop02 이라는 이름이 /etc/hosts파일에 기입되어 있다면 slaves에는 다음과 같이 기입한다

{% highlight bash %}
hadoop02
hadoop03
{% endhighlight %}

#### core-site.xml 파일 수정

HDFS가 사용할 fs.defaultFS만을 설정해 주면 된다.
<name> 태그 사이에는 fs.defaultFS를 , <value> 태그 사이에는 hdfs 주소로 사용할 값을 넣어주면 된다(hdfs://{namenode 호스트명}:{사용할 포트} 형식이다)

{% highlight bash %}
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop01:9000</value>
        </property>
</configuration>
{% endhighlight %}

#### hdfs-site.xml 파일 수정

- dfs.replication 속성 : hdfs에서 관리하는 파일의 복제본 수를 의미한다. 같은 파일을 몇 개의 노드에 복제해 놓을지를 결정해 놓는 수이다.
- dfs.namenode.name.dir 속성 : 파일 시스템 이미지를 저장할 로컬 파일 시스템의 경로이다.
- dfs.namenode.checkpoint.dir : Secondary NameNode에서 체크포인팅 데이터를 저장할 로컬 파일 시스템의 경로이다.
- dfs.namenode.data.dir : HDFS 데이터 블록을 저장할 로컬 파일 시스템의 경로이다.
- dfs.http.address : 웹 HDFS 모니터링 페이지로 사용할 주소이다.

{% highlight bash %}
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>2</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>/data/hadoop/namenode</value>
        </property>
        <property>
                <name>dfs.namenode.checkpoint.dir</name>
                <value>/data/hadoop/namesecondary</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>/data/hadoop/datanode</value>
        </property>
        <property>
                <name>dfs.http.address</name>
                <value>hadoop01:8000</value>
        </property>
</configuration>
{% endhighlight %}

#### mapred-site.xml
{% highlight bash %}
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>


<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>
{% endhighlight %}

#### yarn-site.xml 파일 수정
{% highlight bash %}
<?xml version="1.0"?>
<configuration>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
        <property>
                <name>yarn.nodemanager.local-dirs</name>
                <value>/data/hadoop/yarn-nm-local</value>
        </property>
        <property>
                <name>yarn.resourcemanager.fs.state-store.uri</name>
                <value>/data/hadoop/yarn-rmstore</value>
        </property>
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop01</value>
        </property>
</configuration>
{% endhighlight %}

#### Hadoop 2 실행하기

Hadoop을 처음 실행하기 위해서는(정확히 말해서 hdfs를 처음 시작하기 위해서는) Namenode 포맷이 필요하다.
{하둡 설치 경로}/bin/hdfs namenode -format 명령어를 실행하면, 네임노드에서 사용하는 파일을 저장하는 디렉토리(dfs.namenode.name.dir에서 설정한 값)이 초기화된다.

그 후 {하둡 설치 경로}/sbin/start-hdfs.sh, {하둡 설치 경로}/sbin/start-yarn.sh를 실행하면 네임노드 , 보조 네임 노드, 데이터 노드가 실행되게 된다.
네임노드에서 jps 명령어를 쳤을 때는 NameNode와 ResourceManager, 데이터노드에서 jps 명령어를 쳤을 때는 DataNode와 NodeManager가 존재해야 한다.

정상적으로 실행되지 않을 경우에는 {하둡 설치 경로}/logs 아래에 있는 로그를 확인하여 문제를 확인한 후 실행하도록 한다.
