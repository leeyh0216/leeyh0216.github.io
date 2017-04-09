---
layout: post
title:  "[License] Software의 License"
date:   2017-04-09 21:32:00 +0900
author: leeyh0216
categories: tip license
---

> Software License에 대해서 조사하고, 그간 개발해온 Code에 적용하는 방안을 작성한 글입니다.
더불어 IntelliJ를 사내에서 사용할 수 있는지에 대해서도 조사해보았습니다.

### License

Software License에는 다양한 종류가 있고, 대부분의 작은 회사들에서는 이러한 규칙을 지키지 않고 있지만, 이번 기회를 통해 License를 조사하고, 각 License의 특징을 확인하여 내가 지금까지 사용한 Software 들에 이러한 License를 위반한 사례가 있는지를 확인하려 이 글을 작성하게 되었다.

#### License의 종류

- Apache License : 아파치 소프트웨어 재단에서 자체적으로 만든 소프트웨어에 대한 라이선스 규정이다. 현재 버전은 2.0이며, 누구든 자유롭게 아파치 소프트웨어를 다운받아 부분 혹은 전체를 개인적 혹은 상업적 목적으로 이용할 수 있으며, 재배포 시에도 반드시 원본 소스코드, 수정된 소스코드가 있다는 것을 포함하지 않아도 된다. 단, 아파치 라이선스 2.0을 포함시켜야 하고, 아파치 소프트웨어 재단에 의해 개발된 소스코드라는 것을 알려야 한다.[(출처 : Wikipedia)](https://ko.wikipedia.org/wiki/%EC%95%84%ED%8C%8C%EC%B9%98_%EB%9D%BC%EC%9D%B4%EC%84%A0%EC%8A%A4)<br>[Apache License](https://www.apache.org/licenses/LICENSE-2.0.html) 페이지에 가보면 자세히 적혀 있지만, 가장 중요한 부분만을 나열해보자면 다음과 같다.

  1. Apache License 아래에 존재하는 창작물을 사용했다면, 라이선스의 복사본을 배포해야 한다.
  2. 수정을 가했다면, 어떤 파일을 수정했는지는 명시해야 한다(수정한 파일을 공개할 의무는 없음).
  3. License 사용자는 창작물의 소스 형태에 있던 저작권 공지, 특허 공지, 상표권 공지, 귀속 공지 등을 그대로 유지하여 배포해야 한다.

위 조항들은 대부분 프로젝트에 포함되어 있는 NOTICE 파일들을 추합하여 하나의 NOTICE로 만들어 배포하면 되는 조항이며, 모든 소스코드의 상단에 해당 LICENSE의 내용을 고지해야 한다. 단, 기존 NOTICE를 수정하면 안된다.

이번에 진행하려는 Project에서 IntelliJ를 사용하고 싶었는데, License를 구매해야 하는 것으로 알고 있었다. 하지만 IntelliJ는 2009년 10월 15일 Open Source화 되었다. 이는 Apache License 2.0을 채택하였으므로, Personal, Commercial 모두에서 Community Edition을 사용할 수 있다(Ultimate Edition은 돈내고 사용해야 한다).
따라서 추가 기능이 필요하지 않다면 IntelliJ IDEA Community Edition을 다운로드 받아서 사용하자(기본적인 Java, Groovy, Scala 등을 지원한다. Play, Spring과 같은 Framework를 쉽게 사용할 수 있는 버전이 Ultimate이고, 이또한 쉽게 사용하지 않고 알아서 사용할 것이라면 Community Edition을 사용해도 무방하다).

- GNU(General Public License) License(GPL) : 자유소프트웨어 재단에서 만든 자유 소프트웨어 라이선스이다. 대표적으로 리눅스 커널이 이용하고 있다. 만일 이 허가를 가진 프로그램을 사용하여 새로운 프로그램을 만들게 되면, 파생된 프로그램 또한 같은 카피라이트를 가져야 한다. 다음과 같은 의무들을 지켜야 한다.

  1. 컴퓨터 프로그램을 어떠한 목적으로든지 사용할 수 있다. 다만 법으로 제한하는 행위는 할 수 없다.
  2. 컴퓨터 프로그램의 실행 복사본은 언제나 프로그램의 소스코드와 함께 판매하거나, 소스코드를 무료로 배포해야 한다.
  3. 컴퓨터 프로그램의 소스 코드를 용도에 따라 변경할 수 있다.
  4. 변경된 프로그램의 소스코드 또한 반드시 공개 배포해야 한다.
  5. 변경된 프로그램 역시 똑같은 라이선스를 취해야 한다. 즉, GPL 라이선스를 취해야 한다.

대표적인 GPL 라이선스가 적용된 라이브러리를 찾아보면 MySQL Connector가 있다. 이를 해석해보면, MySQL Connector를 이용해서 작성된 프로그램일 경우 모든 소스코드를 공개해야 한다는 뜻으로 해석 할 수 있다.

 

