---
layout: post
title:  "[Hadoop] HDFS Command"
date:   2017-03-05 02:50:00 +0900
author: leeyh0216
categories: hadoop dev
---

> HDFS 제어 명령어를 정리한 문서입니다.

### HDFS 명령어 목록

Hadoop에서는 사용자가 HDFS를 제어할 수 있도록 다음과 같이 사용할 수 있는 명령어를 제공한다.

{% highlight bash %}
$ hdfs dfs -ls /    //HDFS의 root 디렉토리 조회
{% endhighlight %}

여기서 hdfs는 사실 HADOOP_HOME/bin에 들어있는 hdfs binary file을 실행하는 것이다.
따라서 HADOOP_HOME/bin이 환경변수에 추가되어 있어야 어느 곳에서든 해당 명령어를 실행해도 hdfs 명령어를 실행할 수 있다.

- 만일 해당 환경변수를 설정해주지 않았다면 ~/.bashrc 에 HADOOP_HOME과 PATH 설정을 해주어야 한다.

대부분의 HDFS 명령어는 Linux에서 사용하는 명령어와 비슷하다(POSIX를 따라하려고 애씀)



#### 파일 목록 보기

- ls : 지정한 디렉토리의 파일 정보를 출력한다. 리눅스의 ls 명령어와 동일하다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -ls /
Found 2 items
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp
drwxr-xr-x   - hadoop    supergroup          0 2017-03-05 14:20 /user
{% endhighlight %}

- lsr : 지정한 디렉토리의 하위 파일 및 디렉토리를 Recursive하게 보여준다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -ls -R /
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging
drwxr-xr-x   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history
drwxrwxrwt   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history/done_intermediate
drwxrwx---   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history/done_intermediate/leeyh0216
-rwxrwx---   2 leeyh0216 supergroup      33660 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history/done_intermediate/leeyh0216/job_1488690401643_0001-1488691840727-leeyh0216-word+count-1488691857368-1-1-SUCCEEDED-default-1488691846369.jhist
-rwxrwx---   2 leeyh0216 supergroup        352 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history/done_intermediate/leeyh0216/job_1488690401643_0001.summary
-rwxrwx---   2 leeyh0216 supergroup     117001 2017-03-05 14:30 /tmp/hadoop-yarn/staging/history/done_intermediate/leeyh0216/job_1488690401643_0001_conf.xml
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging/leeyh0216
drwx------   - leeyh0216 supergroup          0 2017-03-05 14:30 /tmp/hadoop-yarn/staging/leeyh0216/.staging
drwxr-xr-x   - hadoop    supergroup          0 2017-03-05 14:20 /user
drwxr-xr-x   - hadoop    supergroup          0 2017-03-01 23:46 /user/hadoop
-rw-r--r--   2 hadoop    supergroup      25804 2017-03-01 23:46 /user/hadoop/test
drwxr-xr-x   - leeyh0216 leeyh0216           0 2017-03-05 14:23 /user/leeyh0216
drwxr-xr-x   - leeyh0216 leeyh0216           0 2017-03-05 14:30 /user/leeyh0216/test
-rw-r--r--   2 leeyh0216 leeyh0216       45619 2017-03-05 14:23 /user/leeyh0216/test/test.txt
-rwxr-xr-x   - leeyh0216 leeyh0216           0 2017-03-05 14:30 /user/leeyh0216/test/wordcount.result
{% endhighlight %}

* Tip : 경로에 Wildcard 사용
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -ls /user/*/*
-rw-r--r--   2 hadoop supergroup      25804 2017-03-01 23:46 /user/hadoop/test
Found 2 items
-rw-r--r--   2 leeyh0216 leeyh0216       45619 2017-03-05 14:23 /user/leeyh0216/test/test.txt
drwxr-xr-x   - leeyh0216 leeyh0216           0 2017-03-05 14:30 /user/leeyh0216/test/wordcount.result
{% endhighlight %}



#### 파일 용량 확인

- du : 지정한 디렉터리나 파일의 사용량을 확인한다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -du /user/leeyh0216
45796  /user/leeyh0216/test
0      /user/leeyh0216/tst
{% endhighlight %}

- dus : 지정한 디렉토리의 모든 용량의 합을 확인한다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -du -s /user/leeyh0216
45796  /user/leeyh0216
{% endhighlight %}



#### 파일 내용 보기

- cat : 지정한 파일의 내용을 화면에 출력한다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -cat /user/leeyh0216/test/test.txt
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf
dlsnflskdnflkn dsfklnsdflk sdfkn lsdfn slkdf skldnflskdnf slknflksndf klsndlfk sndlfknsdfksndlkfnsdklfnskldnf kdnsf kdsnflskdnfksdnf

//tip : hdfs dfs -cat /user/leeyh0216/test/test.txt | vi - 와 같이 사용한다면 출력된 내용을 vi로 열 수 있다.
{% endhighlight %}



#### 디렉토리 생성

- mkdir : 지정한 경로에 디렉토리를 생성한다. 이미 존재하는 디렉토리를 생성하려 할 경우 오류가 발생한다.
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -mkdir /user/leeyh0216/testdir
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -ls /user/leeyh0216
Found 3 items
drwxr-xr-x   - leeyh0216 leeyh0216          0 2017-03-05 14:30 /user/leeyh0216/test
drwxr-xr-x   - leeyh0216 leeyh0216          0 2017-03-05 20:49 /user/leeyh0216/testdir
drwxr-xr-x   - leeyh0216 leeyh0216          0 2017-03-05 14:23 /user/leeyh0216/tst
{% endhighlight %}



#### 파일 복사

- put : 로컬 디렉토리의 파일을 hdfs에 복사한다.
{% highlight bash %}
// hdfs dfs -put {로컬 파일 디렉토리} {복사할 위치- hdfs }
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -put ./testfile.txt /user/leeyh0216/testfile.txt
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -ls /user/leeyh0216/testfile.txt
-rw-r--r--   2 leeyh0216 leeyh0216        299 2017-03-05 20:52 /user/leeyh0216/testfile.txt
{% endhighlight %}



- get : 하둡 디렉토리의 파일(또는 폴더)를 로컬로 가져온다
{% highlight bash %}
leeyh0216@leeyh0216-ubuntu:~$ hdfs dfs -get /user/leeyh0216/testfile.txt ./
{% endhighlight %}



- cp : 하둡 내에서 파일을 복사한다.(리눅스 명령어와 동일하므로 생략)


- mv :  하둡 내에서 파일을 이동한다.(리눅스 명령어와 동일하므로 생략)
