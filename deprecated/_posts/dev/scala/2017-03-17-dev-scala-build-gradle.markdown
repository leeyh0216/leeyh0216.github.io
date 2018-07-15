---
layout: post
title:  "[Scala] Build with gradle"
date:   2017-03-17 23:32:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Gradle을 이용하여 Scala Project를 빌드하는 방법에 대해 서술한 문서입니다.

### Gradle을 이용하여 Scala Project 빌드하기

Gradle은 Java만을 위한 빌드 도구가 아닌, 범용적인 빌드 도구이기 때문에 plugin을 이용해서 각 언어별 빌드를 지원한다.
그렇기 때문에 모든 Java Project는 상단에 apply plugin : 'java'와 같은 키워드가 붙어있음을 알 수 있다.
응용하게 되면, Scala Project를 빌드하기 위해서는 apply plugin: 'scala'를 이용해야 한다.

다음과 같이 build.gradle 파일을 작성하면, gradle을 이용하여 Scala Project를 빌드할 수 있다.

{% highlight groovy %}
apply plugin : 'scala'
apply plugin : 'eclipse'
apply plugin : 'java'

repositories {
	mavenCentral()
	jcenter()
}

dependencies {
	compile 'org.scala-lang:scala-library:2.11.8'
	testCompile org.scalatest:scalatest_2.11.3.0.0'
	testCompile group : 'junit', name: 'junit', version: '4.+'
}
{% endhighlight %}

위 내용으로 build.gradle 을 생성하고, 디렉토리 레이아웃만 맞추어 소스를 작성하면(src/main/scala, src/test/scala 디렉토리 하위에 패키지 및 클래스를 작성하면 된다) gradle build 명령어를 이용하여 jar를 생성할 수 있고, scala -cp 명령어를 이용하여 해당 jar를 실행할 수 있다.
