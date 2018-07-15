---
layout: post
title:  "[Scala] Build"
date:   2017-03-16 21:38:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala Build 과정을 작성한 문서입니다.

### SBT

일반적으로 scala Project를 Build할 때는 SBT(Simple Build Tool)을 사용한다.
java의 경우에는 gradle, maven을 많이 사용하지만 scala에서 해당 Build Tool의 사용률은 20%도 되지 않는다고 한다.
그렇기 때문에 이 문서에서는 SBT를 사용하여 scala Project를 빌드하는 것을 목표로 한다.

### SBT 설치
> 설치 환경 : Ubuntu 14.04 LTS / JDK 1.7.0_121 / Scala 2.11.8

Ubuntu 기본 Repository에 sbt 설치 파일이 올라와 있지 않기 때문에 다음과 같이 source.list를 추가한 후 다운받을 수 있다.

{% highlight bash %}
echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
sudo apt-get update
sudo apt-get install sbt
{% endhighlight %}

위 과정을 거친 후에 sbt를 설치 후 실행하면 sbt 실행에 필요한 라이브러리들을 sbt가 자동으로 다운로드 받게 되고 사용할 수 있게 된다.

{% highlight bash %}
//sbt명령어를 실행하면 자동으로 필요한 라이브러리를 다운로드 받는 과정이 이어진다.
$ sbt
{% endhighlight %}

### SBT Project Directory Layout

SBT Project의 Directory Layout은 다음과 같다.
{% highlight bash %}
build.sbt
project/
	Dependencies.scala
src/
	main/
		resources/
		scala/
		java/
	test/
		resources/
		scala/
		java/
target/
{% endhighlight %}
단순히 sbt 명령어를 사용해서는 위와 같은 Layout이 구성되지 않는다.(target Directory만 생성된다)
가장 쉽게 위와 같은 Directory 구조를 만들 수 있는 방법은 eclipse plugin을 사용하는 방법이다.

일단 Project의 정보를 담고 있는 build.sbt 파일을 작성해야 한다.
build.sbt 파일 내에 들어가는 내용은 name(프로젝트명), version(프로젝트 버전), scalaVersion(프로젝트에서 사용하는 scala version)이다. 다음과 같이 build.sbt를 작성하자.

{% highlight scala %}
name := "SampleScalaProject"

version := "0.1"

scalaVersion := "2.11.8"
{% endhighlight %}

반드시 각 라인 사이에 엔터를 넣어 띄워주어야 한다.
그 후 sbt 명령어를 실행하면 다음과 같은 Directory Layout이 생성된 것을 확인할 수 있다.

{% highlight bash %}
$ sbt
$ ls
-rw-rw-r-- 1 leeyh0216 leeyh0216   73  3월 16 22:05 build.sbt
drwxrwxr-x 3 leeyh0216 leeyh0216 4096  3월 16 22:05 project
drwxrwxr-x 2 leeyh0216 leeyh0216 4096  3월 16 22:05 target
{% endhighlight %}

아직 src 디렉토리가 생성되지 않았다. SBT Project에서는 src 디렉토리 하위에 존재하는 소스파일만을 인식하므로 반드시 Naming Convention을 지켜주어야 하는데, eclipse Plugin은 이러한 Directory Layout을 자동으로 생성해준다.
다음과 같이 project 폴더 아래에 plugins.sbt 파일을 작성 후 sbt eclipse 명령어를 실행하자.

{% highlight bash %}
$ vi project/plugins.sbt

//아래는 plugins.sbt 내용입니다
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "2.4.0")

$ sbt eclipse
{% endhighlight %}

디렉토리 레이아웃을 확인해보면 다음과 같이 src 디렉토리와 하위 디렉토리가 생성된 것을 확인할 수 있다.

{% highlight bash %}
$  ls src/*/*

src/main/java:

src/main/scala:

src/main/scala-2.11:

src/test/java:

src/test/scala:

src/test/scala-2.11:
{% endhighlight %}

위와 같이 구성 한 후 Eclipse 에서 해당 디렉토리를 Import하면 Scala Project를 사용할 수 있다(물론 Eclipse에 scala plugin이 설치되어 있어야 한다)
