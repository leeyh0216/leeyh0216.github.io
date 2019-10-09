---
layout: post
title:  "ElasticSearch + MetricBeat + Kibana로 서버 모니터링하기"
date:   2018-11-19 15:00:00 +0900
author: leeyh0216
categories:
- elasticsearch
---

# 개요

내가 근무하는 팀에서의 프로젝트는 아래와 같이 크게 두 가지로 분류된다.

* Hadoop Cluster에서 동작하는 배치 작업(일, 시간 단위)
* 위의 배치 작업의 메타데이터 및 작업 상태, 의존성 등을 관리하는 웹 서비스(WAS)

Hadoop Cluster의 경우 다른 팀에서 운영을 맡고 있기 때문에 내가 작성한 프로그램이 사용하는 리소스나 코드를 최적화 시켜주면 운영 과정에서는 별다른 문제가 발생하지 않는다.

그러나 웹 서비스의 경우 동작하는 서버를 우리 팀에서 관리하기 때문에 어플리케이션 뿐만 아니라 서버, DB 또한 우리가 직접 관리해주어야 한다.

가끔 서비스에서 문제가 발생할 경우 로그 확인 뿐만 아니라 Zabbix 등을 이용하여 당시 서버의 리소스 상태를 확인하는 경우가 있는데, 정말 개괄적인 지표만을 제공하기 때문에 좀 더 세분화된 지표를 기록해야 하는 상황이 생기고 있다.

Elastic Search 공식 사이트의 [Getting Started With Elastic Stack](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-elastic-stack.html) 글을 보며 설치 및 실행을 테스트해보고자 한다.

# 설치 환경
* Ubuntu 14.04 On Windows(WCL)
* Oracle JDK 1.8.0_181

# Elastic Stack 설치 및 실행

## Elastic Search 6.5.0 다운로드

[ElasticSearch 다운로드 링크(Linux Tar)](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.5.0.tar.gz) 를 통해 6.5.0 버전의 Elastic Search를 다운로드한다.

## Elastic Search tar.gz 파일 압축 풀기

`tar -zxvf elasticsearch-6.5.0.tar.gz` 명령어를 이용해 설치할 경로에 압축을 푼다.

## Elastic Search 실행

`${설치 경로}/bin/elasticsearch -d` 명령어를 통해 Elastic Search를 실행(기본 옵션)한다. -d 옵션은 데몬으로 동작시킨다는 의미이다.

아래와 같이 몇몇 오류 메시지를 내뱉으며 실행되는 것을 확인할 수 있다.(몇몇 WARN 로그가 발생하지만 일단 건너뛰도록 한다)

{% highlight text %}
[2018-11-19T14:03:43,713][WARN ][o.e.b.JNANatives         ] [unknown] unable to install syscall filter:
... 생략
[2018-11-19T14:03:57,629][INFO ][o.e.x.s.a.s.FileRolesStore] [8HVXscX] parsed [0] roles from file [/usr/lib/elasticsearch-6.5.0/config/roles.yml]
[2018-11-19T14:03:58,365][INFO ][o.e.x.m.j.p.l.CppLogMessageHandler] [8HVXscX] [controller/1259] [Main.cc@109] controller (64 bit): Version 6.5.0 (Build 71882a589e5556) Copyright (c) 2018 Elasticsearch BV
[2018-11-19T14:03:59,034][DEBUG][o.e.a.ActionModule       ] [8HVXscX] Using REST wrapper from plugin org.elasticsearch.xpack.security.Security
[2018-11-19T14:03:59,407][INFO ][o.e.d.DiscoveryModule    ] [8HVXscX] using discovery type [zen] and host providers [settings]
[2018-11-19T14:04:00,676][INFO ][o.e.n.Node               ] [8HVXscX] initialized
[2018-11-19T14:04:00,677][INFO ][o.e.n.Node               ] [8HVXscX] starting ...
[2018-11-19T14:04:00,965][INFO ][o.e.t.TransportService   ] [8HVXscX] publish_address {127.0.0.1:9300}, bound_addresses {[::1]:9300}, {127.0.0.1:9300}
...생략
[2018-11-19T14:04:04,245][INFO ][o.e.x.s.t.n.SecurityNetty4HttpServerTransport] [8HVXscX] publish_address {127.0.0.1:9200}, bound_addresses {[::1]:9200}, {127.0.0.1:9200}
[2018-11-19T14:04:04,248][INFO ][o.e.n.Node               ] [8HVXscX] started
[2018-11-19T14:04:05,487][INFO ][o.e.l.LicenseService     ] [8HVXscX] license [0000f9ea-9beb-4be3-a821-a81a1b65d659] mode [basic] - valid
{% endhighlight %}

웹 브라우저를 통해 `http://localhost:9200`에 접근해보면 아래와 같은 페이지가 뜨는 것을 확인할 수 있다.

![Elastic Search Main Page](/assets/elasticsearch/elasticsearch_intro.jpg)

# Kibana 설치 및 실행

## Kibana 다운로드

[Kibana 다운로드 링크(Linux Tar)](https://artifacts.elastic.co/downloads/kibana/kibana-6.5.0-linux-x86_64.tar.gz)를 통해 Kibana 6.5.0 버전을 다운로드한다.

## Kibana 6.5.0 tar.gz 압축 풀기

`tar -zxvf kibana-6.5.0-linux-x86_64.tar.gz` 명령어를 통해 압축을 푼다. Front End 모듈이 굉장히 많아서 압축을 푸는데 한세월 걸린다...

## Kibana 실행

`${설치경로}/bin/kibana serve` 명령어를 통해 Kibana를 실행한다.

아래와 같이 로그가 찍히며 맨 아랫줄에 Server running at http://localhost:5601 과 같이 실행되었다는 것을 확인할 수 있다.

{% highlight text %}
  log   [14:48:38.563] [info][status][plugin:kibana@6.5.0] Status changed from uninitialized to green - Ready
  log   [14:48:38.659] [info][status][plugin:elasticsearch@6.5.0] Status changed from uninitialized to yellow - Waiting for Elasticsearch
  ... 생략
  log   [14:48:53.541] [info][migrations] Creating index .kibana_1.
  log   [14:48:55.060] [info][migrations] Pointing alias .kibana to .kibana_1.
  log   [14:48:55.288] [info][migrations] Finished in 1747ms.
  log   [14:48:55.292] [info][listening] Server running at http://localhost:5601
{% endhighlight %}

`http://localhost:5601`에 접속해보면 아래와 같이 Kibana의 초기 페이지가 보이는 것을 확인할 수 있다.

![Kibana Intro Page](/assets/elasticsearch/kibana_intro.jpg)

# MetricBeat 설치 및 실행

## 개요

ElasticSearch 에는 XXXBeat라는 제품군이 있다. Elastic Search에서는 이들을 통용하여 Beats라고 부르는데, Beats는 서버에 설치되어 서버의 운영 데이터를 Elastic Search로 전송하는(Log Stash로 전송한 뒤 Log Stash가 Elastic Search로 전송하는 방식도 있음) Agent 이다.

Elastic Search에서는 아래와 같은 Beats들을 제공하고 있다.

| Elastic Beats  | To Capture               |
|----------------|--------------------------|
|   Auditbeat    | Audit Data               |
|   Filebeat     |  Log files               |
|   Heartbeat    | Availability Monitoring  |
|   Metricbeat   |  Metric data             |
|   Packetbeat   |  Network Traffic         |

서버 상태만을 모니터랑할 예정이기 때문에 일단 Metricbeat을 설치해보도록 한다.

## MetricBeat 다운로드

[MetricBeat 다운로드 링크(Linux Tar)](https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-6.5.0-linux-x86_64.tar.gz)를 통해 MetricBeat 6.5.0 버전을 다운로드한다.

## MetricBeat 압축 풀기

`tar -zxvf metricbeat-6.5.0-linux-x86_64.tar.gz` 명령어를 통해 압축을 푼다.

## MetricBeat 실행

### MetricBeat의 system metric 기능 활성화

MetricBeat은 몇몇 Metric 수집/전송 기능을 제공하는데, 이 예제에서는 system metric 수집 및 전송 기능을 테스트한다.

`${설치경로}/bin/metricbeat modules enable system` 명령어를 통해 system metric module을 활성화한다.

### MetricBeat Setup

MetricBeat과 연결된 Kibana의 Dashboard를 초기화하는 과정이다.

`${설치경로}/bin/metricbeat setup -e` 명령어를 통해 초기화 할 수 있다.

아래와 같은 로그 메시지가 발생하며 초기화되었다고 성공 메시지로 명령어가 종료된다.

{% highlight text %}
2018-11-19T23:55:19.747+0900    INFO    instance/beat.go:616    Home path: [/usr/lib/metricbeat-6.5.0-linux-x86_64] Config path: [/usr/lib/metricbeat-6.5.0-linux-x86_64] Data path: [/usr/lib/metricbeat-6.5.0-linux-x86_64/data] Logs path: [/usr/lib/metricbeat-6.5.0-linux-x86_64/logs]
2018-11-19T23:55:19.766+0900    INFO    instance/beat.go:623    Beat UUID: d4c0cd03-35c3-4c65-b51d-222021e17837
2018-11-19T23:55:19.769+0900    INFO    [beat]  instance/beat.go:849    Beat info       {"system_info": {"beat": {"path": {"config": "/usr/lib/metricbeat-6.5.0-linux-x86_64", "data": "/usr/lib/metricbeat-6.5.0-linux-x86_64/data", "home": "/usr/lib/metricbeat-6.5.0-linux-x86_64", "logs": "/usr/lib/metricbeat-6.5.0-linux-x86_64/logs"}, "type": "metricbeat", "uuid": "d4c0cd03-35c3-4c65-b51d-222021e17837"}}}
... 생략
2018-11-19T23:55:19.826+0900    INFO    instance/beat.go:302    Setup Beat: metricbeat; Version: 6.5.0
2018-11-19T23:55:22.836+0900    INFO    add_cloud_metadata/add_cloud_metadata.go:319    add_cloud_metadata: hosting provider type not detected.
2018-11-19T23:55:22.885+0900    INFO    elasticsearch/client.go:163     Elasticsearch url: http://localhost:9200
2018-11-19T23:55:22.898+0900    INFO    [publisher]     pipeline/module.go:110  Beat name: leeyh0216-pc
2018-11-19T23:55:22.903+0900    INFO    elasticsearch/client.go:163     Elasticsearch url: http://localhost:9200
2018-11-19T23:55:22.917+0900    INFO    elasticsearch/client.go:712     Connected to Elasticsearch version 6.5.0
2018-11-19T23:55:22.933+0900    INFO    template/load.go:129    Template already exists and will not be overwritten.
Loaded index template
Loading dashboards (Kibana must be running and reachable)
2018-11-19T23:55:22.938+0900    INFO    elasticsearch/client.go:163     Elasticsearch url: http://localhost:9200
2018-11-19T23:55:22.962+0900    INFO    elasticsearch/client.go:712     Connected to Elasticsearch version 6.5.0
2018-11-19T23:55:22.967+0900    INFO    kibana/client.go:118    Kibana url: http://localhost:5601
2018-11-19T23:55:57.005+0900    INFO    instance/beat.go:741    Kibana dashboards successfully loaded.
{% endhighlight %}

### MetricBeat 서비스 실행

위의 두 과정을 통해 MetricBeat의 System 모듈과 MetricBeat과 연결된 Kibana의 Dashboard를 초기화하였다.

이제 MetricBeat을 실행할 차례이다.

아래 명령어를 통해 MetricBeat을 실행한다.

`${설치경로}/bin/metricbeat -e` 명령어를 입력하면 아래와 같은 로그 메시지가 발생하며 MetricBeat이 실행된 것을 확인할 수 있다.

{% highlight text %}
2018-11-19T23:57:08.793+0900    INFO    instance/beat.go:616    Home path: [/usr/lib/metricbeat-6.5.0-linux-x86_64] Config path: [/usr/lib/metricbeat-6.5.0-linux-x86_64] Data path: [/usr/lib/metricbeat-6.5.0-linux-x86_64/data] Logs path: [/usr/lib/metricbeat-6.5.0-linux-x86_64/logs]
2018-11-19T23:57:08.796+0900    INFO    instance/beat.go:623    Beat UUID: d4c0cd03-35c3-4c65-b51d-222021e17837
2018-11-19T23:57:08.799+0900    INFO    [seccomp]       seccomp/seccomp.go:93   Syscall filter could not be installed because the kernel does not support seccomp
2018-11-19T23:57:08.816+0900    INFO    [beat]  instance/beat.go:849    Beat info       {"system_info": {"beat": {"path": {"config": "/usr/lib/metricbeat-6.5.0-linux-x86_64", "data": "/usr/lib/metricbeat-6.5.0-linux-x86_64/data", "home": "/usr/lib/metricbeat-6.5.0-linux-x86_64", "logs": "/usr/lib/metricbeat-6.5.0-linux-x86_64/logs"}, "type": "metricbeat", "uuid": "d4c0cd03-35c3-4c65-b51d-222021e17837"}}}
2018-11-19T23:57:08.817+0900    INFO    [beat]  instance/beat.go:858    Build info      {"system_info": {"build": {"commit": "ff5b9b3db49856a25b5eda133b6997f2157a4910", "libbeat": "6.5.0", "time": "2018-11-09T18:03:04.000Z", "version": "6.5.0"}}}
2018-11-19T23:57:08.818+0900    INFO    [beat]  instance/beat.go:861    Go runtime info {"system_info": {"go": {"os":"linux","arch":"amd64","max_procs":4,"version":"go1.10.3"}}}
2018-11-19T23:57:08.824+0900    INFO    [beat]  instance/beat.go:865    Host info       {"system_info": {"host": {"architecture":"x86_64","boot_time":"2018-11-16T22:41:58+09:00","containerized":true,"name":"leeyh0216-pc","ip":["169.254.103.52/16","fe80::35ce:4940:4154:6734/64","169.254.182.173/16","fe80::8595:86d3:ab93:b6ad/64","172.22.36.145/28","fe80::e0b9:fff1:14a:3de5/64","10.0.75.1/24","127.0.0.1/8","::1/128","192.168.219.103/24","fe80::f571:57f5:e305:e4ec/64","169.254.39.125/16","fe80::64b2:28fe:35ae:277d/64","169.254.189.130/16","fe80::fd70:9829:96ad:bd82/64"],"kernel_version":"4.4.0-17134-Microsoft","mac":["c4:9d:ed:8e:1f:7c","02:50:f2:00:00:01","02:15:06:8b:5d:5c","00:15:5d:db:85:04","c4:9d:ed:8e:1f:7b","c6:9d:ed:8e:1e:7a","c6:9d:ed:8e:1b:7a"],"os":{"family":"debian","platform":"ubuntu","name":"Ubuntu","version":"16.04.3 LTS (Xenial Xerus)","major":16,"minor":4,"patch":3,"codename":"xenial"},"timezone":"DST","timezone_offset_sec":32400}}}
2018-11-19T23:57:08.826+0900    INFO    [beat]  instance/beat.go:894    Process info    {"system_info": {"process": {"capabilities": {"inheritable":null,"permitted":null,"effective":null,"bounding":["chown","dac_override","dac_read_search","fowner","fsetid","kill","setgid","setuid","setpcap","linux_immutable","net_bind_service","net_broadcast","net_admin","net_raw","ipc_lock","ipc_owner","sys_module","sys_rawio","sys_chroot","sys_ptrace","sys_pacct","sys_admin","sys_boot","sys_nice","sys_resource","sys_time","sys_tty_config","mknod","lease","audit_write","audit_control","setfcap","mac_override","mac_admin","syslog","wake_alarm","block_suspend"],"ambient":null}, "cwd": "/usr/lib/metricbeat-6.5.0-linux-x86_64", "exe": "/usr/lib/metricbeat-6.5.0-linux-x86_64/metricbeat", "name": "metricbeat", "pid": 1790, "ppid": 1641, "seccomp": {"mode":""}, "start_time": "2018-11-19T23:57:07.900+0900"}}}
2018-11-19T23:57:08.827+0900    INFO    instance/beat.go:302    Setup Beat: metricbeat; Version: 6.5.0
2018-11-19T23:57:11.839+0900    INFO    add_cloud_metadata/add_cloud_metadata.go:319    add_cloud_metadata: hosting provider type not detected.
2018-11-19T23:57:11.845+0900    INFO    elasticsearch/client.go:163     Elasticsearch url: http://localhost:9200
2018-11-19T23:57:11.871+0900    INFO    [publisher]     pipeline/module.go:110  Beat name: leeyh0216-pc
2018-11-19T23:57:11.909+0900    INFO    [monitoring]    log/log.go:117  Starting metrics logging every 30s
2018-11-19T23:57:11.911+0900    INFO    instance/beat.go:424    metricbeat start running.
...생략
2018-11-19T23:57:13.116+0900    INFO    template/load.go:129    Template already exists and will not be overwritten.
2018-11-19T23:57:13.118+0900    INFO    pipeline/output.go:105  Connection to backoff(elasticsearch(http://localhost:9200)) established
2018-11-19T23:57:41.953+0900    INFO    [monitoring]    log/log.go:144  Non-zero metrics in the last 30s        {"monitoring": {"metrics": {"beat":{"cpu":{"system":{"ticks":480,"time":{"ms":484}},"total":{"ticks":680,"time":{"ms":687},"value":0},"user":{"ticks":200,"time":{"ms":203}}},"handles":{"limit":{"hard":4096,"soft":1024},"open":6},"info":{"ephemeral_id":"f91190da-ef4b-4bcd-9cc3-9b84419376fa","uptime":{"ms":33233}},"memstats":{"gc_next":6331744,"memory_alloc":5023160,"memory_total":17274296,"rss":25976832}},"libbeat":{"config":{"module":{"running":0},"reloads":1},"output":{"events":{"acked":62,"batches":3,"total":62},"read":{"bytes":1902},"type":"elasticsearch","write":{"bytes":45366}},"pipeline":{"clients":6,"events":{"active":0,"published":62,"retry":24,"total":62},"queue":{"acked":62}}},"metricbeat":{"system":{"cpu":{"events":3,"success":3},"filesystem":{"events":1,"failures":1},"fsstat":{"events":1,"failures":1},"load":{"events":3,"success":3},"memory":{"events":3,"success":3},"network":{"events":24,"success":24},"process":{"events":23,"success":23},"process_summary":{"events":3,"success":3},"uptime":{"events":1,"success":1}}},"system":{"cpu":{"cores":4},"load":{"1":0.52,"15":0.59,"5":0.58,"norm":{"1":0.13,"15":0.1475,"5":0.145}}}}}}
{% endhighlight %}

일정 시간마다 맨 아랫줄과 같은 시스템 정보를 Elastic Search로 전송하는 것을 확인할 수 있다.

Kibana의 Dashboard 메뉴를 클릭해보면 아래와 같이 여러 개의 Dashboard가 생성된 것을 확인할 수 있으며

![Kibana System Dashboard Menu](/assets/elasticsearch/kibana_system_dashboard_menu.jpg)

System Dashboard로 들어가보면 아래와 같이 System 상태가 기록되는 것을 확인할 수 있다.

![Kibana System Dashboard](/assets/elasticsearch/kibana_system_dashboard.jpg)

# 정리

다운로드/설치만을 통해 기본 값으로 실행해서 그런지 막힘 없이 진행할 수 있었다.

대시보드의 지표들을 확인해보니 확실히 Zabbix 보다 많은 기능을 제공해주는 것 같긴 하다. 좀 더 기능적인 부분에 대해 확인해본 후 도입을 결정해야할 것 같다.