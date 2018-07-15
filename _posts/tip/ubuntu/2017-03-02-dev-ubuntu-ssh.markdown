---
layout: post
title:  "[Ubuntu] SSH 원격 호스트 자동 로그인 설정"
date:   2017-03-02 00:50:00 +0900
author: leeyh0216
categories: ubuntu tip
---

>Ubuntu 14.04에서 원격 서버에 ssh public key를 등록하여 비밀번호 없이 로그인 하는 방법을 기술합니다. 

### Public, Private Key 생성

일반적으로 SSH는 비밀번호를 이용하여 로그인을 할 수 있지만, Authorized Key를 이용하여 비밀번호 없이 로그인을 할 수 있다.

다음과 같은 방법으로 Private Key와 Public Key를 만들 수 있다.

{% highlight bash %}
$ ssh-keygen -t rsa
{% endhighlight %}

위 명령어를 입력하여 엔터만 누르면, 기본적으로 id_rsa, id_rsa.pub파일이 생기게 된다.

id_rsa는 로그인을 요청하는 쪽에서, id_rsa.pub는 로그인 요청을 받는 서버에서 가지고 있어야 한다.

일반적인 글들에서는 id_rsa.pub 파일을 원격 서버의 ~/.ssh/authorized_keys 마지막에 다음과 같이 추가시키라고 한다.

{% highlight bash %}
$ cat id_rsa.pub >> ~/.ssh/authorized_keys
{% endhighlight %}

하지만, ssh 데몬 설정이 다르게 되어 있는 경우에는 위와 같이 Public Key를 등록해도 비밀번호를 요구하게 된다.

그럴 땐 2개의 파일을 확인해야 한다.

- /etc/ssh/sshd_config 파일
  로그인 요청을 받는 서버에서 확인해야 하는 파일이다.
  PubkeyAuthentication옵션이 주석처리 되어 있거나 no로 되어 있다면 주석을 해제하고 yes로 바꾸어 주자.
  해당 옵션을 Public Key를 이용하여 로그인 하는 것을 허용하는 옵션이다.
  또한 AuthorizedKeysFile이 %h/.ssh/authorized_keys로 설정되어 있는지를 확인하자.
  위 파일이 다른 이름으로 되어 있다면 id_rsa.pub 파일을 해당 파일에 붙여넣어야 한다.

- /etc/ssh/ssh_config 파일
  로그인 요청을 하는 서버에서 확인해야 하는 파일이다.
  IdentityFile 부분이 주석처리 되어있거나, ~/.ssh/id_rsa로 되어 있지 않은가 확인하고, 해당 경로에 실제로 내가 생성한 id_rsa 파일이 존재하는지 확인하자.
  이는 원격 서버 로그인 시 원격 서버에 등록한 나의 Public Key와 결합 할 Private Key의 위치를 기술한 내용이다.

위 두 파일의 설정을 확인하고, 생성한 id_rsa 파일은 ~/.ssh/에 id_rsa.pub 파일은 원격 서버의 authorized_keys에 추가한 뒤 로그인을 시도하자.

만일 정상적으로 로그인 되지 않는다면 ssh -v {로그인 할 ID}@{원격 호스트 명} 명령을 사용하여 어떤 오류가 존재하는지, 어떤 파일을 사용하여 인증을 시도하는 지 확인해 보자. 
