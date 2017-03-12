---
layout: post
title:  "[Hadoop] Yarn Programming(1)"
date:   2017-03-12 01:39:00 +0900
author: leeyh0216
categories: dev hadoop yarn
---

> 이 문서는 Yarn Application 내부 동작에 대해 확인을 위하여 작성하였습니다. 

### Yarn Application 개발

#### build.gradle 설정

기본적으로 해당 프로젝트는 Hadoop 2.7.3 기준이며 Build Tool은 Gradle 2.14를 사용합니다.
build.gradle 내에는 다음 5개의 Dependency를 설정해줍니다.

- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-common', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-client', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-yarn-api', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-common', version: '2.7.3'
- compile group: 'org.apache.hadoop', name: 'hadoop-hdfs', version: '2.7.3' 

추가적으로 Logging과 Guava Library를 사용하기 위하여 다음 Dependency도 추가해줍니다.
- compile group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.1'
- compile group: 'ch.qos.logback', name: 'logback-core', version: '1.2.1'  
- compile group: 'com.google.guava', name: 'guava', version: '21.0'

#### Yarn Client 초기화

Yarn에서 동작하는 Application을 실행하기 위해서는 리소스 매니저에게 Application Master 실행 요청을 해야합니다.
org.apache.hadoop.yarn.client.api.YarnClient 클래스에서 해당 작업을 담당합니다.

{% highlight java %}
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.yarn.client.api.YarnClient;
import org.apache.hadoop.yarn.client.api.YarnClientApplication;

//YarnClient 객체 초기화
YarnClient yarnClient = YarnClient.createYarnClient();

//Hadoop Configuration 설정 로드, hdfs-site.xml, core-site.xml, yarn-site.xml, mapred-site.xml 파일을 로드해야 한다고 가정합니다.
Configuration conf = new YarnConfiguration();
conf.addResource("/home/leeyh0216/hadoop/etc/hadoop/hdfs-site.xml");
conf.addResource("/home/leeyh0216/hadoop/etc/hadoop/core-site.xml");
conf.addResource("/home/leeyh0216/hadoop/etc/hadoop/yarn-site.xml");
conf.addResource("/home/leeyh0216/hadoop/etc/hadoop/mapred-site.xml");

//Yarn Client에 Hadoop Configuration 세팅
yarnClient.init(conf);

//YarnClient 객체로부터 새로 실행할  YarnApplication 객체를 생성한다.
YarnClientApplication clientApplication = yarnClient.createApplication();

//리소스 매니저로부터 새로운 Application 실행 요청 후 결과를 리턴받는다.
GetNewApplicationResponse response = clientApplication.getNewApplicationResponse();
logger.info("Received Application ID : {}",response.getApplicationId().getId());

int maxMem = response.getMaximumResourceCapability().getMemory();
int maxCores = response.getMaximumResourceCapability().getVirtualCores();
logger.info("Yarn Env > Max Memory : {}, Max Cores : {}",maxMem,maxCores);

{% endhighlight %}

위 과정은 클라이언트에서 Yarn의 ResourceManager에게 새로운 Application 생성 요청을 하고, Resource Manager에서는 새로운 Application을 위한 Application ID를 발급해 주는 코드이다.
Resource Manager는 해당 과정에서 아직 새로 실행할 Application에게 어떠한 자원도 할당하지 않고, Application Master 실행 시 설정해야 할 Application ID(Integer)와 현재 Yarn 설정(최대 사용 가능한 Memory, Cpu Core 수 등)을 리턴한다.

...(작성중)

