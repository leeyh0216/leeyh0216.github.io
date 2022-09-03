---
layout: post
title:  "Apache Spark과 S3 Compatible Object Storage 연동 시 Custom Endpoint 이슈"
date:   2022-09-03 15:05:00 +0900
author: leeyh0216
tags:
- apache-spark
- s3
- hadoop
---

# Apache Spark과 S3 Compatible Object Storage 연동 시 Custom Endpoint 이슈

사내에서 개발하는 시스템에서 Apache Spark과 S3 Compatible Object Storage인 Ceph를 연동해야 할 일이 생겼다.

Ceph는 S3 Compatible한 Gateway를 제공하기 때문에 Apache Spark 실행 시 `spark.hadoop.fs.s3a.endpoint`에 해당 Gateway의 주소를 설정해주고, Access Key와 Secret Key까지 설정해주면 금방 될 거라 생각했다. [Databricks 블로그](https://docs.databricks.com/data/data-sources/aws/amazon-s3.html)에도 동일 방식으로 설명하고 있었기 때문이다.


## 이슈 상황

우선 테스트한 환경은 다음과 같다.

* 네트워크: 인터넷과의 연결이 차단된 폐쇄망 환경
* Ceph: 버전은 정확히 알지 못함
* Apache Spark: 3.1.2
* Apache Hadoop: 2.6.0 & 3.2.0

호기롭게 `spark.hadoop.fs.s3a.endpoint`에 Ceph Gateway Endpoint를 설정하고 실행을 했는데, Ceph에 Write하는 과정에서 다음과 같은 오류가 발생하였다.

```
Caused by: org.apache.http.conn.ConnectTimeoutException: Connect to s3.amazonaws.com ...
```

위 오류가 발생했을 때 처음 든 생각은 "`spark.hadoop.fs.s3a.endpoint` 옵션을 잘못 주었나?" 였다. 그러나 코드를 보기 시작하니 내 잘못이 아니라는 걸 느꼇다.

## 문제 해결

우선 나는 AWS를 현업에서 사용해본 적이 없다. 그러나 문제 해결에 도움이 되었던 점은 동일 프로젝트의 서버 사이드에서 AWS Java SDK를 사용하였으므로, 그 경험을 기반으로 문제 해결을 시작했다.

만일 이 경험이 없었다면 Hadoop S3 FileSystem 관련 코드를 한줄한줄 파고 들어가야 했을 것이고, 문제를 해결하는데 오래 걸렸을 것 같다.

### AWS Java SDK에서 S3 연결 시 Custom Endpoint를 설정하는 법

AWS Java SDK에서 S3 Client를 사용할 때 Custom Endpoint를 설정하기 위해서는 `AmazonS3ClientBuilder`의 `withEndpointConfiguration`을 호출하여 자체 Endpoint 정보를 넘겨주면 된다.

```
// S3 client
final AmazonS3 s3 = AmazonS3ClientBuilder.standard()
    .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(endPoint, regionName))
    .withCredentials(new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKey, secretKey)))
    .build();
```

`EndpointConfiguration` 객체를 생성할 때 `endPoint` 위치에 Object Storage의 Gateway 주소를 넘겨주어야만 S3 API 호출 시 해당 Endpoint로 Routing한다.

### `S3AFileSystem`에서 `AmazonS3Client` 초기화 부분 찾아보기

Spark에서 S3에 데이터를 쓸 때 사용하는 FileSystem 클래스는 `S3AFileSystem` 클래스이다. 우선 Hadoop 2.6.0 에서 수행했기 때문에, https://github.com/apache/hadoop/blob/release-2.6.0/hadoop-tools/hadoop-aws/src/main/java/org/apache/hadoop/fs/s3a/S3AFileSystem.java 의 코드를 확인하였다.

```
ClientConfiguration awsConf = new ClientConfiguration();
awsConf.setMaxConnections(conf.getInt(MAXIMUM_CONNECTIONS, 
    DEFAULT_MAXIMUM_CONNECTIONS));
awsConf.setProtocol(conf.getBoolean(SECURE_CONNECTIONS, 
    DEFAULT_SECURE_CONNECTIONS) ?  Protocol.HTTPS : Protocol.HTTP);
awsConf.setMaxErrorRetry(conf.getInt(MAX_ERROR_RETRIES, 
    DEFAULT_MAX_ERROR_RETRIES));
awsConf.setSocketTimeout(conf.getInt(SOCKET_TIMEOUT, 
    DEFAULT_SOCKET_TIMEOUT));

s3 = new AmazonS3Client(credentials, awsConf);
```

핵심적으로 보아야 할 부분은 `s3` 변수(AmazonS3Client 객체)를 초기화하는 부분인데, 위와 같이 구성되어 있다. 초기화 시 사용자가 전달한 설정들도 반영해야 할 것 같은데, 자체적으로 관리하는 변수들(MAXIMUM_CONNECTIONS, SECURE_CONNECTIONS 등)을 제외하고는 반영하는 부분이 없다.

이 포인트에서 "아, Hadoop 2.6.0의 문제인가? 어차피 Hadoop 3으로 넘어가는 상황이니 Hadoop 3.2.2 버전의 코드를 확인해보자" 라는 생각이 들었다. Hadoop 2.6 버전대와 동일하게 `S3AFileSystem`을 보면 되는데, `s3`를 초기화할 때 다음과 같이 Reflection을 사용하고 있다.

```
s3 = ReflectionUtils.newInstance(s3ClientFactoryClass, conf)
        .createS3Client(name, bucket, credentials);
```

우선 S3 Client Factory를 생성한 뒤, 해당 Factory의 `createS3Client`를 호출하여 `AmazonS3Client` 객체를 초기화하는 것을 확인할 수 있다. 해당 Factory 클래스는 [`DefaultS3ClientFactory`](https://github.com/apache/hadoop/blob/rel/release-3.2.2/hadoop-tools/hadoop-aws/src/main/java/org/apache/hadoop/fs/s3a/DefaultS3ClientFactory.java) 였고, `createS3Client` 메서드를 확인해보면 아래와 같다.

```
@Override
public AmazonS3 createS3Client(URI name,
    final String bucket,
    final AWSCredentialsProvider credentials) throws IOException {
        Configuration conf = getConf();
        final ClientConfiguration awsConf = S3AUtils.createAwsConf(getConf(), bucket);
        return configureAmazonS3Client(
            newAmazonS3Client(credentials, awsConf), conf);
}
```

여기서 주의깊게 봐야 할 메서드는 `configureAmazonS3Client` 였는데, 해당 메서드의 내용은 아래와 같다.

```
private static AmazonS3 configureAmazonS3Client(AmazonS3 s3,
      Configuration conf)
      throws IllegalArgumentException {
    String endPoint = conf.getTrimmed(ENDPOINT, "");
    if (!endPoint.isEmpty()) {
      try {
        s3.setEndpoint(endPoint);
      } catch (IllegalArgumentException e) {
        String msg = "Incorrect endpoint: "  + e.getMessage();
        LOG.error(msg);
        throw new IllegalArgumentException(msg, e);
      }
    }
    return applyS3ClientOptions(s3, conf);
}
```

우리가 원했던 Endpoint 설정을 해주는 것을 확인할 수 있다. 드디어 문제가 해결된 것일까?

### Hadoop 3.2.0 버전의 이슈

Apache Spark은 보통 Hadoop 버전에 맞게 Pre built 되어 빌드된 tarball을 제공하고 있다. 팀 내에서 Apache Spark 3.1.2 + Hadoop 3.2 Prebuilt를 사용하고 있었고, 3.2.0 브랜치에서 Endpoint를 잘 넣어주는 것을 확인했기에 다시 프로그램을 돌려보았다.

```
Exception in thread "init" java.lang.NoSuchMethodError: org.apache.hadoop.util.SemaphoredDelegatingExecutor.<init>(Lcom/google/common/util/concurrent/ListeningExecutorService;IZ)V
    at org.apache.hadoop.fs.s3a.S3AFileSystem.create(S3AFileSystem.java:769)
    at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:1169)
    at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:1149)
    at org.apache.hadoop.fs.FileSystem.create(FileSystem.java:1108)
    at org.apache.hadoop.fs.FileSystem.createNewFile(FileSystem.java:1413)
    at org.apache.accumulo.server.fs.VolumeManagerImpl.createNewFile(VolumeManagerImpl.java:184)
    at org.apache.accumulo.server.init.Initialize.initDirs(Initialize.java:479)
    at org.apache.accumulo.server.init.Initialize.initFileSystem(Initialize.java:487)
    at org.apache.accumulo.server.init.Initialize.initialize(Initialize.java:370)
    at org.apache.accumulo.server.init.Initialize.doInit(Initialize.java:348)
    at org.apache.accumulo.server.init.Initialize.execute(Initialize.java:967)
    at org.apache.accumulo.start.Main.lambda$execKeyword$0(Main.java:129)
    at java.lang.Thread.run(Thread.java:748)
```

흠... 뭔가 문제가 있는 모양이라 검색해보니 Hadoop Jira [HADOOP-16080](https://issues.apache.org/jira/browse/HADOOP-16080)으로 해당 이슈가 리포팅되어 있었다.

Affects Versions가 3.2.0이고, Fix Versions가 3.2.2였다. Fetch 버전이 올라간거고 클라이언트 이슈이기 때문에 설치한 Spark Client Home의 jar 디렉토리의 Hadoop 관련 라이브러리 버전을 3.2.0 -> 3.2.2로 변경해주었다.(덤으로 Guava도 14 -> 27로 바꾸어주어야 하는데, 이 때가 가장 떨렸다. Guava는 가장 충돌이 많이 났던 라이브러리라서...)

### 해결 완료 및 회고

위의 사항들을 모두 적용한 뒤 정상적으로 Object Storage를 통한 Read와 Write가 수행되는 것을 확인하였다.

생각보다 Hadoop과 Object Storage 연동 관련 공식 문서가 부실했고, 버전에 따른 이슈들도 있어서 해결하는데에 하루 정도 걸렸던 것 같다. 좋은 경험이었지만 다시는 경험하고 싶지 않은 이슈였다. 