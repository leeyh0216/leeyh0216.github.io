---
layout: post
title:  "Spring + MongoDB + Docker 조합 사용 테스트"
date:   2019-05-06 10:00:00 +0900
author: leeyh0216
tags:
- docker
- spring
---

# 프로젝트 초기화

## git 초기화

1. Git 페이지에서 [spring_mongodb_docker](https://github.com/leeyh0216/spring_mongodb_docker) Repository를 초기화한다.

2. ```git pull https://github.com/leeyh0216/spring_mongodb_docker.git``` 명령어를 통해 로컬로 Clone 한다.

3. [gitignore.io](https://gitignore.io) 페이지에서 gradle, java, intellij로 초기화한 ```.gitignore```을 디렉토리에 추가한다.

## Spring Project 초기화

1. 최상위 디렉토리 아래에 spring-boot-test 라는 이름으로 디렉토리를 생성한다.

2. spring-boot-test에서 ```gradle init``` 명령어로 gradle 프로젝트를 초기화한다.

3. build.gradle을 아래와 같이 작성한다.[Spring Boot Guide 페이지 참고](https://spring.io/guides/gs/spring-boot/)

{% highlight groovy %}
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:2.0.5.RELEASE")
    }
}

apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

repositories {
    mavenCentral()
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    testCompile("junit:junit")
}
{% endhighlight %}

# Spring Application 코드 작성

## Hello World!

간단하게 http://localhost:8080/hello로 접속하면 Hello World!를 출력하는 프로그램을 작성하였다.

{% highlight java %}
package com.leeyh0216.test;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class SampleServer {

    public static void main(String[] args) throws Exception {
        SpringApplication.run(SampleServer.class, args);
    }

    @GetMapping("/hello")
    public String hello(){
        return "Hello World";
    }
}
{% endhighlight %}

별다른 설정 없이 띄웠으므로 http://localhost:8080/hello로 접속 시 브라우저에 Hello World가 출력되는 것을 볼 수 있다.

## Spring Boot Data MongoDB Starter Dependency 추가

Maven Repository에서 Spring Boot Data MongoDB Starter Dependency를 추가한다.

[Maven Repository: Spring Boot Data MongoDB Starter](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-data-mongodb) 페이지에서 사용하는 Spring에 맞는 버전을 적절히 골라 build.gradle의 dependencies에 추가한다.

## CRUD Service, Controller, Entitry 객체 클래스 구현

아래와 같이 간단한 CRUD 서비스를 구현한다.

### Person.java
{% highlight java %}
package com.leeyh0216.test.crud;

public class Person {

    private String id = "";

    public String name = "";

    public int age = 20;

    //Getter, Setter 생략
}
{% endhighlight %}

### CRUDService.java
{% highlight java %}
package com.leeyh0216.test.crud;

import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class CRUDService {

    private MongoTemplate mongoTemplate;

    public CRUDService(MongoTemplate mongoTemplate){
        this.mongoTemplate = mongoTemplate;
    }

    public Person createPerson(Person person){
        mongoTemplate.insert(person);
        return person;
    }

    public List<Person> getPeople(){
        return mongoTemplate.findAll(Person.class);
    }
}
{% endhighlight %}

### CRUDController.java

{% highlight java %}
package com.leeyh0216.test.crud;

import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/person")
public class CRUDController {

    private CRUDService crudService;

    public CRUDController(CRUDService crudService) {
        this.crudService = crudService;
    }

    @PostMapping("")
    public Person createPerson(@RequestBody Person person){
        return crudService.createPerson(person);
    }

    @GetMapping("")
    public List<Person> getPeoples(){
        return crudService.getPeople();
    }
}
{% endhighlight %}

## Dockerfile 작성

프로젝트 빌드 후 build/libs 하위에 spring-boot-test.jar이 생성되기 때문에, JDK 8 이미지에서 해당 JAR 파일을 실행시키는 형태로 작성하였다.

{% highlight bash %}
FROM openjdk:8-jre-alpine

RUN mkdir -p /data/spring-boot-test
COPY ./build/libs/spring-boot-test.jar /data/spring-boot-test/

ENTRYPOINT ["java", "-jar", "/data/spring-boot-test/spring-boot-test.jar"]
EXPOSE 8080
{% endhighlight %}

## Docker 이미지 빌드 및 실행

Spring 프로젝트 디렉토리에서 ```docker build --tag spring-boot-test:0.1 .``` 명령어를 통해 Dockerfile을 빌드하여 이미지를 생성해냈다.

이후 ```docker run -d --name spring-boot-test -p 8080:8080 spring-boot-test:0.1``` 명령어를 이용하여 이미지를 컨테이너로 만들어 실행시켰다.

http://localhost:8080/hello로 접속 시 정상적으로 동작하는 것을 확인할 수 있었다.

## Docker Swarm 의 서비스로 실행하기

spring-boot-test를 단일 Docker Container가 아닌 Docker Swarm의 서비스로 만들어보자.

사실상 위의 명령어와 별다를게 없다. ```docker run -d``` 명령어만 ```docker service create``` 명령어로 바꾸어주면 된다.

```docker service create --name spring-boot-test -p 8080:8080 spring-boot-test:0.1``` 명령어를 이용하여 서비스로 실행시켰으며, 아래와 같이 정상적으로 서비스 등록이 된 것을 볼 수 있었다.

```
image spring-boot-test:0.1 could not be accessed on a registry to record
its digest. Each node will access spring-boot-test:0.1 independently,
possibly leading to different nodes running different
versions of the image.

cri8td3bfy6d3z8u342dr3ju3
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

```docker service ls``` 명령어를 이용하여 확인했을 때도 아래와 같이 서비스 기동 현황을 확인할 수 있었다.

```
ID                  NAME                MODE                REPLICAS            IMAGE                  PORTS
i78wqngny6hp        apacheserver        replicated          1/1                 apache2:0.1            *:81->80/tcp
cri8td3bfy6d        spring-boot-test    replicated          1/1                 spring-boot-test:0.1   *:8080->8080/tcp
```

# MongoDB Service

위에서 실행한 서버의 EndPoint를 호출해보면, /hello 경로는 정상적으로 호출되는 것을 확인할 수 있으나, /person 경로는 아래와 같이 오류가 발생하는 것을 확인할 수 있다.

```
{"timestamp":"2019-05-06T04:24:20.077+0000","status":500,"error":"Internal Server Error","message":"Timed out after 30000 ms while waiting to connect. Client view of cluster state is {type=UNKNOWN, servers=[{address=mongodb:27017, type=UNKNOWN, state=CONNECTING, exception={com.mongodb.MongoSocketException: mongodb: Name does not resolve}, caused by {java.net.UnknownHostException: mongodb: Name does not resolve}}]; nested exception is com.mongodb.MongoTimeoutException: Timed out after 30000 ms while waiting to connect. Client view of cluster state is {type=UNKNOWN, servers=[{address=mongodb:27017, type=UNKNOWN, state=CONNECTING, exception={com.mongodb.MongoSocketException: mongodb: Name does not resolve}, caused by {java.net.UnknownHostException: mongodb: Name does not resolve}}]","path":"/person"}
```

이는 위의 Spring 서버에서 참조하는 MongoDB 인스턴스가 존재하지 않기 때문이다.

## Mongodb Instance를 Docker Service로 실행

[DockerHub: Mongo](https://hub.docker.com/_/mongo) 페이지를 참고하여 Docker Image를 Pull 한다.

3.6 Image를 받을 예정이며, 아래 명령어를 사용하면 된다.

```
docker pull 3.6
```

아래와 같이 Image가 Pull 되는 것을 확인할 수 있다.

```
3.6: Pulling from library/mongo
7e6591854262: Pull complete
089d60cb4e0a: Pull complete
9c461696bc09: Pull complete
45085432511a: Pull complete
e5182dfcfa20: Pull complete
ccb099326ee3: Pull complete
75804f28c4b1: Pull complete
765a10b214be: Pull complete
36cbec4a23a5: Pull complete
8d7c112fee50: Pull complete
22a72bf1a592: Pull complete
7c24e128abe6: Pull complete
44337c6f0bee: Pull complete
Digest: sha256:cb57ecfa6ebbefd8ffc7f75c0f00e57a7fa739578a429b6f72a0df19315deadc
Status: Downloaded newer image for mongo:3.6
```

위의 이미지를 Docker Service로 실행시킨다.

```
docker service create --name mongodb -p 27017:27017 mongo:3.6
```

아래와 같이 정상적으로 MongoDB Service가 실행된 것을 확인할 수 있다.

```
xnrho4emt1cwem3fsyiadd3yb
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

telnet을 이용하여 27017 포트에 접속되는지를 확인해보았다.

```
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

그러나 ```docker service log spring-boot-test``` 명령어를 실행해보면 아직 MongoDB Instance를 찾을 수 없다는 오류가 발생하고 있는 것을 확인할 수 있다.

```
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    | 2019-05-06 04:31:10.649  INFO 1 --- [}-mongodb:27017] org.mongodb.driver.cluster               : Exception in monitor thread while connecting to server mongodb:27017
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    | com.mongodb.MongoSocketException: mongodb
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.ServerAddress.getSocketAddress(ServerAddress.java:188) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.connection.SocketStreamHelper.initialize(SocketStreamHelper.java:59) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.connection.SocketStream.open(SocketStream.java:57) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.connection.InternalStreamConnection.open(InternalStreamConnection.java:126) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.connection.DefaultServerMonitor$ServerMonitorRunnable.run(DefaultServerMonitor.java:114) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at java.lang.Thread.run(Thread.java:748) [na:1.8.0_201]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    | Caused by: java.net.UnknownHostException: mongodb
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at java.net.InetAddress.getAllByName0(InetAddress.java:1281) ~[na:1.8.0_201]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at java.net.InetAddress.getAllByName(InetAddress.java:1193) ~[na:1.8.0_201]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at java.net.InetAddress.getAllByName(InetAddress.java:1127) ~[na:1.8.0_201]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at java.net.InetAddress.getByName(InetAddress.java:1077) ~[na:1.8.0_201]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      at com.mongodb.ServerAddress.getSocketAddress(ServerAddress.java:186) ~[mongodb-driver-core-3.6.4.jar!/:na]
spring-boot-test.1.rqxbxzydhp3z@linuxkit-00155d249100    |      ... 5 common frames omitted
```

# Docker Overlay Network 구성

위에서 spring-boot-test 서버가 MongoDB Instance를 찾을 수 없는 이유는 간단하다. mongodb라는 Host가 어디에도 등록되어 있지 않기 때문이다.

이를 찾을 수 있는 방법은 2가지가 있다.

1. spring-boot-test의 ```application.properties``` 파일에 있는 MongoDB Host를 IP로 설정하거나, host 파일에 등록하는 방법

2. Docker의 Overlay Network를 이용하는 방법

여기서는 Docker의 Overlay Network를 이용해 보도록 한다.

## Overlay Network 생성

아래 명령어를 이용하여 ```backend```라는 이름의 Docker Overlay Network를 생성한다.

```
docker network create --attachable --driver overlay backend
```

아래 명령어를 통해 정상적으로 네트워크가 생성되었는지 확인한다.

```
docker network ls
```

```
NETWORK ID          NAME                DRIVER              SCOPE
7p056vdum6up        backend             overlay             swarm
df830a7a0306        bridge              bridge              local
054011e7ede2        docker_gwbridge     bridge              local
c2fb0615051d        host                host                local
j39apemc19b0        ingress             overlay             swarm
1d10ebc33b36        none                null                local
```

위와 같이 NAME이 backend인 Overlay Network가 생성된 것을 확인할 수 있다.

## Service를 Overlay Network에 연결

일단 위에 실행했던 spring-boot-test와 mongodb 서비스를 모두 삭제한다.

```
docker service rm spring-boot-test
docker service rm mongodb
```

새롭게 spring-boot-test와 mongodb 서비스를 실행할 때는 ```--network backend``` 옵션을 주고 실행한다.

```
docker service create --name spring-boot-test -p 8080:8080 --network backend spring-boot-test:0.1
```

```
image spring-boot-test:0.1 could not be accessed on a registry to record
its digest. Each node will access spring-boot-test:0.1 independently,
possibly leading to different nodes running different
versions of the image.

olcbruuu5jutlqg6vofqpzslz
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

```
docker service create --name mongodb -p 27017:27017 --network backend mongodb:3.6
```

```
z5z3crpdcy416mco2bpawwnbr
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

## 테스트

spring-boot-test 서버를 통해 person을 생성하고 조회해보도록 하자.

일단 http://localhost:8080/person 을 조회해보자.

```
curl http://localhost:8080/person
[]
```

위와 같이 빈 JSON Array가 반환되는 것을 알 수 있다.

이제 person을 생성해보도록 한다.

```
curl -XPOST -H "Content-Type: application/json" -d '{"name":"leeyh0216","age":28}' http://localhost:8080/person

{"id":"","name":"leeyh0216","age":28}
```

생성된 Person이 반환되는 것을 확인할 수 있다.

이제 person 목록을 조회해보도록 한다.

```
curl http://localhost:8080
[{"id":"","name":"leeyh0216","age":28}]
```

JSON Array에 우리가 만든 leeyh0216 이름을 가진 Person 객체가 반환되는 것을 확인할 수 있다.

# 결론

Docker Swarm은 생각보다 사용하기 쉽게 만들어진 것을 알 수 있었다.

물론 아직 Dockerfile 을 최적화해서 만든다거나, Docker Swarm의 Network 구성 등을 좀 더 알아봐야 할 필요가 있지만, 현재의 Dedicated 된 서버보다 훨씬 효율적으로 운영이 가능할 것으로 보인다.