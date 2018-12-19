---
layout: post
title:  "Inversion of Control Containers and the Dependency Injection pattern"
date:   2018-12-19 10:00:00 +0900
author: leeyh0216
categories: Spring IoC DI
---

> 이 글은 Martin Fowler의 Inversion of Control Containers and the Dependency Injection pattern을 요약 정리한 글입니다.

# Inversion of Control Containers and the Dependency Injection pattern

많은 오픈소스들은 J2EE 기술에 대한 대안을 구축하는 다양한 활동을 하고 있다. 이러한 활동은 J2EE의 복잡도를 획기적인 방법으로 낮추기 위함이다.

이들이 다루는 공통적인 이슈는 서로 다른 요소들을 어떻게 결합하는지에 대한 것이다. 이러한 문제를 해결하기 위해 많은 프레임워크들이 등장했고, 몇몇 프레임워크는 다른 레이어에 있는 요소(컴포넌트)들을 조합하는 방식을 제공한다. 이러한 방식은 Lightweight Container(경량화된 컨테이너)라고 불리운다.

## Component and Service

* 컴포넌트: 어플리케이션에서 사용될 목적으로 만들어진 소프트웨어 구성 요소이며, 컴포넌트는 컴포넌트 작성자가 허용하는 방식으로만 확장이 가능하다.

* 서비스: 컴포넌트와 비슷하지만, 외부 어플리케이션에서 사용한다는 점에서 차이를 보인다. 

컴포넌트는 Jar, Assembly, DLL 등을 통해 제공되어 로컬에서 사용하며, 서비스는 웹서비스, 메시징 시스템, RPC, Socket 등의 원격 인터페이스를 통해 사용된다.

# A Naive Example

원본 글에서 아래와 같은 예제가 제공된다.

{% highlight java %}
class MovieLister{

    public Movie[] moviesDirectedBy(String arg) {
        List allMovies = finder.findAll();
        for(Iterator it = allMovies.iterator(); it.hasNext();) {
            Movie movie = (Movie)it.next();
            if (!movie.getDirector().equals(arg)) it.remove();
        }
    }
    return (Movie[])allMovies.toArray(new Movie[allMovies.size()]);
}
{% endhighlight %}

영화 목록 중 인자로 주어진 값을 영화 감독으로 가지는 영화를 추출하여 반환하는 `moviesDirectedBy` 라는 메서드가 정의되어 있는 `MovieLister`라는 클래스를 제공하고 있다.

위 예제의 요점은 '**MovieLister 객체와 Finder 객체를 연결하는 법**'이다. `MovieLister` 객체는 영화가 어떻게 저장되는지에 대해서는 전혀 관심이 없다. 단지 조건에 부합하는 영화를 `moviesDirectedBy` 메서드로 찾고 싶을 뿐이다. 즉, `Finder` 객체와의 의존성을 없애고 싶은 것이다.

이러한 문제를 해결하기 위해 `Finder` 객체를 아래와 같이 `Interface`로 정의하였다.
{% highlight java %}
public interface MovieFinder {
    List findAll();
}
{% endhighlight %}

`Finder`를 Interface로 정의하므로써, `Finder`와 `MovieLister` 객체의 분리는 잘 된 것 같다. 그러나 아래와 같이 `MovieLister` 클래스의 생성자에서 `Finder` 객체를 상속한 `ColonDelimitedMovieFinder` 클래스의 객체를 명시적으로 할당해준다면 어떨까?

{% highlight java %}
class MovieLister {
    private MovieFinder finder;
    public MovieLister() {
        this.finder = new ColonDelimitedMovieFinder("movies1.txt");
    }
}
{% endhighlight %}

코드의 작성자만 사용할 때는 문제가 없지만, 다른 사람이 위 클래스를 사용하게 된다면

* ColonDelimitedMovieFinder 클래스의 생성자로 전달되는 "movies1.txt" 변경 불가
* 텍스트 파일이 아닌 XML 혹은 SQL 등에 영화 목록이 저장되어 있을 경우 사용 불가

와 같은 문제점에 부딪히게 된다.

![Dependency Issue](/assets/spring/both_dependency.jpeg)

위 그림에서와 같이 `MovieLister`가 `MovieFinder` Interface에 의존하지만, 동시에 이를 상속하는 Concrete Class인 `MovieFinderImpl`을 생성하므로써 둘 모두에 의존성이 생기게 된다. 즉, `MovieFinder`를 Interface로 만든 보람이 없어지게 된다. `MovieLister`와 `MovieFinder`를 제대로 Decouple 시키기 위해서는, 아래 `MovieFinderImpl`에 대한 의존성을 완전히 없애야 한다.

이러한 문제는 Plugin 패턴으로 해결할 수 있고, 이러한 Plugin을 어플리케이션에 조합해서 넣는 방법을 IoC를 사용하므로써 해결할 수 있다.

### Plugin 이란?

![Plugin](/assets/spring/plugin.jpg)

Interface는 어플리케이션 코드가 서로 다른 구현을 필요로 하는 여러 런타임 환경에서 동작할 때 주로 사용된다. 개발자는 Interface를 적절히 구현하여 Factory Method에 제공(리턴)한다.

예를 들어 위 그림에서 DomainObject 객체가 Primiary Key를 생성해야 할 때, Unit Test 환경에서는 In-Memory Counter를 사용하고, Production 환경에서는 Database Managed Sequence를 사용해야 한다.

만일 IdGenerator를 제공하는 Factory Method를 직접 구현한다면, 조건문을 사용해서 실행 환경에 따라 일일이 다른 IdGenerator 구현체를 반환해야 할 것이다. 이러한 경우 실행 환경, 배포 환경 등이 바뀔 때마다 모든 Factory 의 조건문을 수정해야하는 상황이 발생할 수 있다.

Plugin은 중앙화된 런타임 구성을 제공하므로써 이러한 문제를 해결한다.

## IoC(Inversion of Control)

위의 A Naive Exmple 에서 `MovieLister` 클래스는 `MovieFinder`를 상속한 구현체를 생성자에서 인스턴스화하였다. 이는 `MovieFinder`가 Plugin 화 될 수 없게 만든다. 이를 가능하게 하기 위해서는, 별도의 Assembly 