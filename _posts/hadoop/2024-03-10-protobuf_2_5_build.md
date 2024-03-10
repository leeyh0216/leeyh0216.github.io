---
layout: post
title:  "protobuf 2.5 빌드하기(Apple Silicon)"
date:   2024-3-10 12:10:00 +0900
author: leeyh0216
tags:
- hadoop
---

> 이 문제는 Apple Silicon(M1, M2, M3 등)에서만 발생합니다. Ubuntu나 Intel Mac에서는 발생하지 않을 수 있음을 유의하시기 바랍니다.

Hadoop 3.2.2 버전을 빌드하려다보니, 아래와 같은 메시지가 발생하며 빌드에 실패하였다.

```
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.2.2:protoc (compile-protoc) on project hadoop-yarn-api: org.apache.maven.plugin.MojoExecutionException: 'protoc --version' did not return a version -> [Help 1]
```

`protoc`가 없어서 발생한 문제였고, `protoc`를 포함하는 `protobuf`를 설치하려고 brew를 뒤져보니, 지원하는 버전이 `3.20.3`, `21.12` 밖에 없었다.

`protocolbuffers/protobuf` Repository에는 모든 버전이 존재할 터이니 빌드를 해서 사용하려고 했는데, 9년 전에 릴리즈된 버전이었고 왠지 이런저런 이유로 빌드가 잘 되지 않을 것 같았다.

# 빌드하기

## 사전 작업 

우선 `protocolbuffers/protobuf` Repository를 Clone하고, 2.5.0 버전의 tag로 변경한다.

```
$ git clone https://github.com/protocolbuffers/protobuf.git
$ cd protobuf
$ git checkout v2.5.0
```
<!-- 
그리고 빌드를 하기 위해서는 `make`, `automake`, `autoconf` 의존성을 먼저 설치해야 한다.

```
$ brew install make automake autoconf
``` -->

## 빌드

### `autogen.sh` 실행

`README.md`에는 나와 있지 않지만, 프로젝트 최상위 경로의 `autogen.sh`를 실행해야 한다. `autogen.sh`는 `configure`를 환경에 맞게 생성해주는 스크립트이다.

```
./autogen.sh

Google Test not present.  Fetching gtest-1.5.0 from the web...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1586  100  1586    0     0    454      0  0:00:03  0:00:03 --:--:--   455
tar: Error opening archive: Unrecognized archive format
```

설치 실패가 나는데, 이는 `autogen.sh`의 22번째 라인에서 실행되는 `curl`이 실패했기 때문이다.

```
curl http://googletest.googlecode.com/files/gtest-1.5.0.tar.bz2 | tar jx
```

`tar jx` 명령어를 제외하고 `curl`만 해봤을 때, 해당 파일이 존재하지 않는다는 페이지가 뜬다. 

다행히도 Fedora Repository에 `gtest-1.5.0` 버전이 있기 때문에, 아래 링크로 대체한 뒤 다시 `autogen.sh`을 실행하면 해당 오류는 발생하지 않는 것을 확인할 수 있다.

--- 

이후에 다음 오류가 발생할 수 있다.(사용자 환경에 따라 의존 라이브러리가 설치되어 있는 경우 발생하지 않을 수 있음)

```
./autogen.sh: line 38: autoreconf: command not found
```

`autogen.sh`는 `autoconf`에 의존성을 가지고 있기 때문에, `autoconf`를 설치해준 뒤, 다시 `autogen.sh`를 실행하도록 한다.

```
$ brew install autoconf
$ ./autogen.sh
```

---

이후에 다음 오류가 발생할 수 있다.(이 또한 사용자 환경에 따라 발생하지 않을 수 있음)

```
autoreconf: error: aclocal failed with exit status: 2
```

`autoconf`가 다시 `automake`에 의존성을 가지고 있어 발생한 문제로, `automake`을 설치해준 뒤, 다시 `autogen.sh`를 실행하도록 한다.

```
$ brew install automake
$ ./autogen.sh
```

이 단계가 완료되면 현재 경로에 `configure` 바이너리가 생성된 것을 확인할 수 있다.

### `configure` 실행

`configure`를 실행한다. 이 단계에서는 별다른 오류는 발생하지 않는다.

```
$ ./configure
```

### `make` 실행

`make` 명령어 실행 시, 다음과 같은 오류가 발생하는 것을 확인할 수 있다.

```
$ make

...
./google/protobuf/stubs/atomicops_internals_macosx.h:163:41: error: unknown type name 'Atomic64'; did you mean 'Atomic32'?
                                        Atomic64 increment) {
                                        ^~~~~~~~
                                        Atomic32
./google/protobuf/stubs/atomicops.h:65:15: note: 'Atomic32' declared here
typedef int32 Atomic32;
              ^
fatal error: too many errors emitted, stopping now [-ferror-limit=]
9 warnings and 20 errors generated.
make[2]: *** [atomicops_internals_x86_gcc.lo] Error 1
make[1]: *** [all-recursive] Error 1
make: *** [all] Error 2
```

정확한 이유는 나와있지 않지만, [Develop on Apple Silicon (M1) macOS](https://cwiki.apache.org/confluence/display/HADOOP/Develop+on+Apple+Silicon+%28M1%29+macOS) 문서에도 2.5.0 버전의 protobuf가 Apple Silicon을 지원하지 않기 때문에 다른 버전의 protobuf를 사용할 수 있도록 수정해서 빌드하라고 되어 있다.

그러나 굳이 그럴 필요 없이 누군가 해당 문제를 수정한 Patch를 올려 놨기 때문에, 이를 통해 패치 진행 후 빌드하면 된다.(https://gist.github.com/liusheng/64aee1b27de037f8b9ccf1873b82c413)

```
$ curl -L -O https://gist.githubusercontent.com/liusheng/64aee1b27de037f8b9ccf1873b82c413/raw/118c2fce733a9a62a03281753572a45b6efb8639/protobuf-2.5.0-arm64.patch
$ patch -p1 < protobuf-2.5.0-arm64.patch
$ ./configure --disable-shared
$ make
```

위 명령어를 실행하면 src 경로에 protoc가 생성된 것을 확인할 수 있다.

해당 바이너리를 /usr/local/bin에 옮기고 다시 Hadoop을 빌드하면, 정상적으로 빌드되는 것을 확인할 수 있다.