---
layout: post
title:  "Spring cloud zuul(1)"
date:   2018-11-10 15:00:00 +0900
author: leeyh0216
categories: microservice spring-cloud gateway
---

# 개요

2017년 후반부터 2018년 초까지 팀 내 서비스들을 마이크로서비스 아키텍쳐 형태로 개발하는 프로젝트를 진행하였다.

사내에서 L7 Switch를 제공하고 있었지만, 서비스가 추가될 때마다 요청하기도 번거롭고 Software Level Gateway에서만 할 수 있는 작업들도 여럿 있었다.

당시에 Gateway 후보로 Spring Cloud Zuul를 검토했었는데, 아래 이유들 때문에 도입하지 않았었다.

* Spring Cloud Zuul이 사용하는 Service Discovery는 Spring Cloud Eureka 혹은 Properties 파일이다.
* Spring Cloud Eureka를 사용하려면 Git Server가 필요하다. 팀 내에서 Git Server를 운영하고 있지 않았고, 인프라 추가는 제한되어 있는 상태였다.
* Properties 파일은 서버 추가 시 Gateway 가 구동되고 있는 서버들의 모든 파일을 일일히 수정해 주어야 한다.(물론 Jenkins를 통해 배포할 수도 있었다.)

사실 위 이유들보다는 사용해보지 않았던 서비스이기 때문에 문제가 생겼을 때 Trouble Shooting에 대한 두려움이 가장 컸던 것 같다.

그래서 Spring boot와 Okhttp를 이용해서 경량화된 Gateway를 구현해서 사용하고 있다.

하지만 몇 달간 운영을 해보니 아래와 같은 점을 느끼게 되었다.

* 개발 초기를 제외하고는 서비스 추가가 잦지 않다.
* 인하우스로 개발한 Gateway의 기능이 부족하다(인증, 인가, 모니터링, Custom).

그래서 이번에 Spring Cloud Zuul을 테스트해보고 팀 내에 Gateway 변경을 건의할 예정이다.

# Spring Cloud Zuul

Zuul은 원래 Netflix에서 사용하던 Software level Gateway이며, 아래와 같은 기능을 제공한다.(출처: https://github.com/Netflix/zuul)

* Dynamic Routing
* Monitoring
* Resiliency
* Security
* etc...

spring.io Guide 문서인 Routing-and-Filtering을 이용하여 데모를 만들어 보았다.(https://github.com/leeyh0216/spring-cloud-zuul-sample)

## Project 초기화

Spring Cloud Zuul 서버와 샘플 서버 하나를 만들 예정이다.

아래와 같이 settings.gradle과 build.gradle을 작성했다.

### settings.gradle
{% highlight groovy %}
rootProject.name = 'spring-cloud-zuul-sample'
include 'zuul-server', 'sample-server
{% endhighlight %}

### build.gradle
{% highlight groovy %}
buildscript {
    ext {
        springBootVersion = '1.4.6.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'idea'
    apply plugin: 'spring-boot'
    apply plugin: 'io.spring.dependency-management'

    sourceCompatibility = 1.8
    targetCompatibility = 1.8

    group 'com.leeyh0216'
    version '1.0'

    repositories {
        mavenCentral()
    }

    dependencies {
        compile group: 'org.springframework.boot', name: 'spring-boot-starter-web', version: '1.4.6.RELEASE'
        testCompile group: 'org.springframework.boot', name: 'spring-boot-starter-test', version: '1.4.6.RELEASE'
    }
}

project(":zuul-server") {
    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:Brixton.SR5"
        }
    }

    dependencies {
        compile group: 'org.springframework.cloud', name: 'spring-cloud-starter-zuul', version: '1.4.6.RELEASE'
    }
}

project(":sample-server") {

}
{% endhighlight %}

## Sample Server 작성

1 ~ 2개 정도의 GET Method로 구성된 Sample Server를 하나 만들 예정이다.

아래와 같이 SampleServer 클래스를 작성하였다.

{% highlight java %}
package com.leeyh0216.sampleserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class SampleServer {

    public static void main(String[] args) {
        SpringApplication.run(SampleServer.class, args);
    }

    @GetMapping("/")
    public String helloworld(){
        return "Hello World";
    }

    @GetMapping("/bye")
    public String bye(){
        return "Good Bye";
    }
}
{% endhighlight %}

작성 후 http://localhost:8080으로 들어가본 결과 아래와 같이 정상적으로 Hello World가 출력되는 것을 확인할 수 있었다.

![Sample Server Hello World](/assets/microservice/sampleserver.jpg)

## Zuul Server 작성

이제 Sample Server로의 Routing을 담당할 Zuul Server를 만들어보자.

아래와 같이 ZuulServer 클래스를 만든다. Annotation 넣는 것 빼고는 해줄게 없다.

{% highlight java %}
package com.leeyh0216.zuulserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableZuulProxy
@SpringBootApplication
public class ZuulServer {

    public static void main(String[] args){
        SpringApplication.run(ZuulServer.class, args);
    }
}
{% endhighlight %}

이제 ZuulServer 기동에 필요한 application.yml을 작성해야 한다. application.yml 에는 아래와 같은 설정들이 들어간다.

### server.port
Spring Boot Web application이 사용할 포트이다.

위에서 Sample Server가 8080(기본 포트)를 사용하기 때문에 Zuul은 8081을 사용하도록 설정한다.

### ribbon.eureka.enabled
Spring cloud zuul은 내부적으로 Client Side Loadbalancing에 Ribbon을 사용하고, Ribbon은 다시 Eureka를 사용할 수 있다고 한다.

우리는 일단 Eureka를 사용하지 않을 것이므로 false로 설정한다.

### zuul.routes.sample.url
Spring cloud zuul은 Web Request를 등록된 서비스에 Routing 한다.

서비스들은 zuul.routes 프로퍼티 아래에 추가될 수 있다. 만일 sample 서비스를 추가한다고 하면 zuul.routes.sample과 같이 표현할 수 있다.

이제 zuul 서버(http://localhost:8081)로 들어오는 요청 중 첫번째 Path가 sample인 요청들은 모두 sample 서비스로 라우팅된다.

zuul이 해당 요청을 sample 서비스의 어떤 URL로 라우팅할지 결정하는 프로퍼티가 zuul.routes.{서비스명}.url 이다.

위에서 만든 sample server가 http://localhost:8080에 떠 있기 때문에, zuul.routes.sample.url은 http://localhost:8080으로 설정한다.

이제 http://localhost:8081/sample 혹은 http://localhost:8081/sample/**에 매핑되는 요청들은 모두 http://localhost:8080 아래로 매핑된다.

### application.yml
만들어진 application.yml은 아래와 같다.

{% highlight yaml %}
server:
  port: 8081

ribbon:
  eureka:
    enabled: false

zuul:
  routes:
    sample:
      url: http://localhost:8080
{% endhighlight %}

이제 아래와 같이 http://localhost:8081/sample 을 들어가보면, Sample Server의 /로 라우팅되어 Hello World가 출력되는 것을 볼 수 있다.

![Zuul Server Hello World](/assets/microservice/zuulserver.jpg)

http://localhost:8081/sample/bye로 들어가보면 Sample Server의 /bye로 라우팅 되어 Good Bye가 출력되는 것을 볼 수 있다.

![Sample Server Good Bye](/assets/microservice/zuulserver_goodbye.jpg)

## 동작 테스트

Spring Cloud Zuul의 몇몇 Properties를 변경 적용해보고, 어떻게 동작하는지 테스트해보자.

### 목적지 서버가 다운된 경우

목적지 서버에 문제가 생겨 다운된 경우는 어떻게 동작할까?

위에서 띄워 놓은 Sample Server를 Kill 한 뒤, http://localhost:8081/sample로 접속해보았다.

![Sample Server Dead](/assets/microservice/sampleserver_dead.jpg)

위와 같이 Status Code 500이 반환되며 localhost:8080에서 Connection Refused가 반환되었다고 표현되는 것을 볼 수 있다.(사내에서 구현한 경량 Gateway의 경우 503 Service Temporarily Unavailable을 반환하도록 해놓았다)

![Zuul Default Timeout](/assets/microservice/sampleserver_dead_default_timeout.jpg)

크롬 개발자모드 Network 탭에서는 응답 시간이 약 2.05초 정도 걸린 것으로 확인되었다.

Zuul에서는 zuul.host.connect-timeout-millis 옵션을 사용해서 목적지 서버에 대한 Connection Timeout을 설정할 수 있다.

zuul.host.connect-timeout-millis를 100으로 설정하고 테스트 해보았다.

![Zuul Short Timeout](/assets/microservice/sampleserver_dead_short_timeout.jpg)

위와 같이 약 220밀리초 정도 소요된 것을 확인할 수 있었다.

### 목적지 서버의 응답이 늦는 경우

목적지 서버에 문제가 생겨 응답이 늦어지는 경우를 가정해보자.

하염없이 응답을 기다리기 보다 Timeout을 설정하여 예상 응답 시간을 초과하면 다른 서버로 재 요청을 하는 등의 대안을 고려해볼 수 있다.

Zuul에서는 zuul.host.socket-timeout-millis 옵션을 사용해서 목적지 서버에 대한 Socket Timeout(Read Timeout)을 설정할 수 있다.

일단 아래와 같이 Sample Server에서 / 경로에 대한 응답을 5초 후에 하도록 코드를 변경한다.

{% highlight java %}
    @GetMapping("/")
    public String helloworld(){
        //임의로 응답을 5초 지연시킴
        try {
            Thread.sleep(5000);
        }
        catch(Exception e){
            e.printStackTrace();
        }
        return "Hello World";
    }
{% endhighlight %}

![Sample Server Slow Response](/assets/microservice/sampleserver_slow_response.jpg)

위와 같이 응답 시간이 5초 이상으로 증가한 것을 확인할 수 있다.

이제 Zuul 서버에서 zuul.host.socket-timeout-millis 값을 4000으로 설정하고 http://localhost:8081/sample로 접속해보도록 하자.

Sample Server 에서의 5000ms 이후에 응답할 것이기 때문에, http://localhost:8081/sample에서는 설정에 따라 500 에러와 함께 Read Timeout 오류가 발생해야 한다.

![Zuul Server Read Timeout](/assets/microservice/zuulserver_read_timeout.jpg)

위와 같이 정상적으로 Read Timeout이 발생하는 것을 확인할 수 있었다.

### 2개 이상의 서버를 목적지 서버로 활용하는 방법

위의 예제에서는 Sample Service에 매핑되어 있는 URL이 http://localhost:8080 하나였다.

보통 High Availability 구성을 하면 서비스 별로 2개 이상의 서버를 가질 수 있기 때문에, zuul.routes.{서비스명}.url 에 여러 개의 URL을 넣을 수 있어야 한다.

이 경우 Ribbon을 사용해야 한다고 나와 있다. Ribbon은 Client Side LoadBalancer인데, 이 내용은 다른 글에서 다루도록 하겠다.

Ribbon은 이미 spring-cloud-starter-zuul 의존성을 추가하므로써 자동으로 Import 되기 때문에 별도로 Dependency를 추가하지 않아도 된다.

Ribbon을 이용하여 여러 개의 목적지 서버를 사용하기 위해서는 Zuul Server의 application.yml 파일을 수정해주면 된다.

zuul:
  routes:
    sample:
      path: /sample/**
      serviceId: sample-service

#### zuul.routes.{서비스명}.path

Zuul 서버로 들어온 요청의 Path와 zuul.routes.{서비스명}.path 의 패턴이 일치하면 해당 서비스로 요청이 라우팅된다.

예를 들어, zuul.routes.sample-service.path: /sample/** 으로 설정하면 http://localhost:8081/sample/**로 들어온 요청은 sample-service로 라우팅된다.

#### zuul.routes.{서비스명}.serviceId

Ribbon을 사용하기 위해 서비스의 ID를 지정해준다. {서비스명}에 들어갈 값과 달라도 되지만, 이왕이면 일치시켜주는 것이 좋을 것 같다.

#### {서비스ID}.ribbon.listOfServers

zuul.routes.{서비스명}.path의 패턴과 일치하는 요청이 들어왔을 때 라우팅한 URL 목록이다.

처음 zuul.routes.{서비스명}.url에는 1개만 넣을 수 있었는데 여기에는 여러 개의 URL을 등록할 수 있다.

#### Zuul Server application.yml

{% highlight yaml %}
server:
  port: 8081

ribbon:
  eureka:
    enabled: false

sample-service:
  ribbon:
    listOfServers: http://localhost:8080, http://localhost:8082

zuul:
  routes:
    sample:
      path: /sample/**
      serviceId: sample-service
{% endhighlight %}

위와 같이 Zuul Server의 application.yml을 수정해 보았다.

Sample Server를 8080과 8082로 띄울 것이므로 sample-service.ribbon.listOfServers 에 http://localhost:8080과 http://localhost:8082를 기재하였다.

#### 테스트

8080으로 띄우는 Sample Server의 / 페이지에서는 "Hello World", 8082에서는 "Hello World!!" 를 출력하도록 수정하고 서버를 구동하였다.

위의 application.yml을 적용한 Zuul Server 또한 띄운 후, http://localhost:8081/sample-service로 접속해 보았다.

![Zuul Server Ribbon Sample1](/assets/microservice/zuulserver_ribbon_sample1.jpg)

![Zuul Server Ribbon Sample2](/assets/microservice/zuulserver_ribbon_sample2.jpg)

위와 같이 특정 호출은 "Hello World"가 출력되고 다른 호출은 "Hello World!!"가 출력되는 것을 볼 수 있다.

8080 포트로 띄운 Sample Server를 Kill 한 후, 접속을 시도해보았다.

![Zuul Server Ribbon Sample2](/assets/microservice/zuulserver_ribbon_sample3.jpg)

위와 같이 Status Code 500과 함께 TIMEOUT이라는 메시지가 반환되는 것을 확인할 수 있다.

Zuul Server의 로그에서는 com.netflix.zuul.exception.ZuulException: Forwarding error 라는 오류 메시지와 함께 아래와 같은 StackTrace를 확인할 수 있었다.

{% highlight text %}
com.netflix.zuul.exception.ZuulException: Forwarding error
	at org.springframework.cloud.netflix.zuul.filters.route.RibbonRoutingFilter.handleException(RibbonRoutingFilter.java:158) ~[spring-cloud-netflix-core-1.1.5.RELEASE.jar:1.1.5.RELEASE]
	at org.springframework.cloud.netflix.zuul.filters.route.RibbonRoutingFilter.forward(RibbonRoutingFilter.java:133) ~[spring-cloud-netflix-core-1.1.5.RELEASE.jar:1.1.5.RELEASE]
... 생략

Caused by: com.netflix.hystrix.exception.HystrixRuntimeException: sample-service timed-out and no fallback available.
	at com.netflix.hystrix.AbstractCommand$21.call(AbstractCommand.java:783) ~[hystrix-core-1.5.3.jar:1.5.3]
	at com.netflix.hystrix.AbstractCommand$21.call(AbstractCommand.java:768) ~[hystrix-core-1.5.3.jar:1.5.3]
{% endhighlight %}

HystrixRuntimeException에 의해 발생하였으며 sample-service가 Timout 되었고 가능한 Fallback 메소드가 없다고 나와 있다.(Hystrix도 나중에 다루어볼 예정이다)

다시 8080 포트로 띄우지 않을 경우에는 호출 시 계속 동일한 오류가 발생한다.

특정 서버가 다운되었을 때, 리스트에 존재하는 다른 서버로 라우팅하는 방법은 다음 글에서 다루도록 한다.

# 정리

Netflix Wiki나 Spring Cloud Netflix Documentation에 정리가 잘 되어 있어 사용이 쉬웠고, application.yml 설정만으로도 기본적인 기능은 사용이 가능한 것이 좋았다.

다만, 단순 라우팅이 아닌 다른 기능을 사용하려면 Ribbon, Eureka, Hystrix 등을 알아야 하고, 설정이 무지하게 많은게 좀 걱정이다.

버전도 워낙 많아서 대응하기 어려울 것 같긴 하지만, 한번 제대로 알아놓으면 추후에 인하우스 게이트웨이를 전환할 때 요긴하게 쓸 수 있을 것 같다.