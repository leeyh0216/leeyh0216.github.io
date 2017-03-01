---
layout: post
title:  "[Docker] Getting Startted"
date:   2017-03-01 16:00:00 +0900
categories: docker dev
---

> Docker 기본 사용법을 서술한 문서입니다.
Ubuntu 14.04 에서 Ubuntu 14.04 Container를 띄우는 방법을 서술합니다.

### Ubuntu 14.04 이미지 다운로드

Docker Container를 띄우기 위해서는 기본 이미지가 필요합니다.
Docker Hub 에 올라와 있는 Ubuntu 이미지를 다운로드 받습니다.

{% highlight bash %}
$ sudo docker search ubuntu //Docker Hub에 Ubuntu를 포함하는 Image 검색

$ sudo docker pull ubuntu

{% endhighlight %}

### Docker 기본 명령어

- Docker Container 실행

  Docker Image를 기반으로 새로운 Container를 실행합니다.

{% highlight bash %}
$ sudo docker run -i -t --name {컨테이너 이름} ubuntu //ubuntu 이미지를 {컨테이너 이름} 으로 시작합니다.
{% endhighlight %}

  -i 옵션의 경우 interactive 의 줄임말로써, 실행된 Container와 상호작용(입력/출력)이 가능하게 하라는 옵션이며, -t옵션의 경우 새로운 session으로 Container에 붙겠다는 의미입니다.
  --name은  실행될 Container에 이름을 붙이는 옵션입니다.

- Docker Image 목록 확인

  다운로드 한 Image 목록을 확인합니다.

{% highlight bash %}
$ sudo docker images
{% endhighlight %}

- Docker Container 목록 확인

  모든 Container 목록을 확인합니다.

{% highlight bash %}
$ sudo docker ps -a
{% endhighlight %}

  -a 옵션을 붙이지 않고 실행하면, 실행 중인 Container 목록만을 확인할 수 있습니다.

- Docker Container에서 빠져나오기

  Ctrl+p, Ctrl+q 버튼을 연이어 누르면 컨테이너에서 빠져나올 수 있습니다.

- Docker Container에 재접속하기

  실행 중인 Container에 다시 접속하는 방법은 다음과 같습니다.

{% highlight bash %}
$ sudo docker attach {컨테이너 이름 또는 ID}
{% endhighlight %}
