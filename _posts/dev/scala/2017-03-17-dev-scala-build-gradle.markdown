---
layout: post
title:  "[Scala] Build with gradle"
date:   2017-03-17 23:32:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Gradle을 이용하여 Scala Project를 빌드하는 방법에 대해 서술한 문서입니다.

### Gradle을 이용하여 Scala Project 빌드하기

#### Gradle Scala Project Layout
다음은 Gradle을 사용했을 때 Scala Project의 Directory Layout이다.
SBT 때와 다른 점은 build.sbt 파일이 build.gradle로 변경되었다는 것 뿐이다.

{% highlight bash %}

/src
	/main
		/java
		/scala
		/resources
	/test
		/java
		/scala
		/resources

{% endhighlight %}
