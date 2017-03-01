---
layout: post
title:  "[Docker] How to install Docker"
date:   2017-03-01 15:24:00 +0900
categories: docker dev
---

> Ubuntu 14.04 기준 Docker 설치법에 대한 글을 작성합니다. [Docker 공식 홈페이지](https://docs.docker.com/engine/installation/linux/ubuntu)를 참조하였습니다.

### Docker Repository 추가하기

  Docker 최신 버전은 우분투 Repository에 등록되어 있지 않기 때문에, Docker에서 운영하는 Repository를 참조하여 최신 버전을 다운받을 수 있다.

- https 지원 패키지 설치(Docker의 Repository가 https로 되어있기 때문)
{% highlight bash %} 
  $ sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
{% endhighlight %}
- curl로 key 정보를 다운로드 받은 다음 apt-key에 추가
{% highlight bash %}
  $ curl -fsSL https://apt.dockerproject.org/gpg | sudo apt-key add -
{% endhighlight %}
-  Docker 공식 Repository를 Repository 목록에 추가
{% highlight bash %}
  $ sudo add-apt-repository "deb https://apt.dockerproject.org/repo/ ubuntu-$(lsb_release -cs) main"
{% endhighlight %}

### Docker 설치

  Repository 추가가 끝났으므로, apt-get install 명령어를 입력하여 Docker를 설치할 수 있다.

{% highlight bash %}
  $ sudo apt-get update
  $ sudo apt-get install docker-engine
{% endhighlight %}
