---
layout: post
title:  "싱글톤 패턴"
date:   2021-04-11 13:00:00 +0900
author: leeyh0216
tags:
- designpattern
---

# 싱글톤 패턴

싱글톤 패턴은 프로세스(Java에서는 JVM) 내에 1개의 클래스 인스턴스만을 갖도록 보장하고, 이에 대한 전역 접근점을 제공하는 패턴이다.

## 단순하게 구현하기(Eager Initialization Singleton)

싱글톤 패턴을 가장 단순하게 구현하는 방법은 외부에서 해당 클래스의 인스턴스를 생성할 수 없도록 private 생성자를 만들고, 1개의 인스턴스만을 초기화(public static final)하여 이에 대한 접근을 제공하는 방식이다.

{% highlight java %}
public class SimpleSingleton {
  //public static final으로 1개의 인스턴스만을 초기화하여 접근점을 제공하고, 2개 이상의 인스턴스를 못 만들게 한다.
  public static final SimpleSingleton INSTANCE = new SimpleSingleton();

  private SimpleSingleton(){
    //생성자는 private으로 만들어 외부에서 새로운 인스턴스를 만들 수 없도록 한다.
  }
}
{% endhighlight %}

많은 글에서 "싱글턴 객체 사용 유무와 관계 없이 클래스가 로딩되는 시점에 항상 싱글톤 객체가 생성되어 메모리를 소모하기 때문에 비효율적이다" 라고 기술되어 있고 실제로도 맞는 이야기이다. 다만, 자바의 Lazy Class Loading 방식에 대한 이해가 없는 상태에서는 "클래스가 로딩되는 시점"에 대해 잘못 이해할 수 있다.

> 자바의 Class Loading에 관해서는 [지난번 글](https://leeyh0216.github.io/2020-04-18/java_class_loader)을 읽어보면 좋고, Class Loading 시점에 관해서는 [When a class is loaded and initialized in JVM - Java](https://javarevisited.blogspot.com/2012/07/when-class-loading-initialization-java-example.html) 글을 참고하면 된다.
>
> 아래 케이스들 모두 위 글을 토대로 테스트한 내용이다.

[When a class is loaded and initialized in JVM - Java](https://javarevisited.blogspot.com/2012/07/when-class-loading-initialization-java-example.html)에서 확인할 수 있듯, Java에서 Class가 Loading되는 경우는 아래와 같다.

* `new`를 통한 초기화
* `class.forName()`과 함께 Reflection을 사용하는 경우
* 해당 클래스의 static 메소드가 실행된 경우
* 해당 클래스의 static 필드에 값을 할당 혹은 사용하는 경우(단, static final 제외)
* Top-Level Class인데, assert 문을 사용하는 경우 

이 중 첫번째(`new`는 private method로 막아두었음)와 마지막 케이스를 제외한 세 가지 경우에 대해 ClassLoading이 발생하고 Singleton 객체가 초기화되는지 확인해보자.

### 해당 클래스에 아무런 접근이 없는 경우

**Singleton.java**
{% highlight java %}
public class Singleton {
  public static final Singleton INSTANCE = new Singleton();

  private Singleton() {
    System.out.println("Simple singleton instance initialized.");
  }
}
{% endhighlight %}

**Test.java**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
  }
}
{% endhighlight %}

**실행 결과**
```
Hello world
```

위 `Test` 클래스에서는 `Singleton` 클래스에 대한 어떠한 접근도 없기 때문에, ClassLoader는 `Singleton` 클래스를 로드하지 않는다. 따라서 `Singleton` 클래스의 객체는 생성되지 않기 때문에 "Simple singleton instance initialized" 메시지는 출력되지 않는다.

### `class.forName()`과 함께 Reflection을 사용하는 경우

**Singleton.java**
{% highlight java %}
public class Singleton {
  public static final Singleton INSTANCE = new Singleton();

  private Singleton() {
    System.out.println("Simple singleton instance initialized.");
  }
}
{% endhighlight}

**Test.java**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
    Class.forName("com.leeyh0216.test.Singleton");
  }
}
{% endhighlight %}

**실행 결과**
```
Hello world
Simple singleton instance initialized.
```

위 `Test` 클래스에서는 `Class.forName` 메서드를 통해 강제로 ClassLoading을 유발한다. 따라서 `Singleton` 객체가 초기화되며 "Simple singleton instance initialized" 메시지가 출력된다.

### 해당 클래스의 static 메소드가 실행된 경우

**Singleton.java**
{% highlight java %}
public class Singleton {
  public static final Singleton INSTANCE = new Singleton();

  private Singleton() {
    System.out.println("Simple singleton instance initialized.");
  }

  public static void func() {
    System.out.println("This is static method");
  }
}
{% endhighlight %}

**Test.java**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
    Singleton.func();
  }
}
{% endhighlight %}

**실행 결과**
```
Hello world
Simple singleton instance initialized.
This is static method
```

위 `Test` 클래스에서는 `Singleton` 클래스의 정적 메소드인 `func`를 호출한다. 정적 메소드 호출 시에는 ClassLoading이 발생하기 때문에 `func` 메소드 실행 전에 `Singleton` 객체가 초기화되며 "Simple singleton instance initialized" 메시지가 호출되는 것을 확인할 수 있다.

### 해당 클래스의 static 필드의 값을 할당 혹은 사용하는 경우

**Singleton.java**
{% highlight java %}
public class Singleton {
  public static final Singleton INSTANCE = new Singleton();
  public static int VALUE = 1;

  private Singleton() {
    System.out.println("Simple singleton instance initialized.");
  }
}
{% endhighlight %}

**Test.java(할당)**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
    Singleton.VALUE = 2;
  }
}
{% endhighlight %}

**Test.java(사용)**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
    System.out.println(Singleton.VALUE);
  }
}
{% endhighlight %}

위 두 케이스 모두 ClassLoading이 발생한다. 다만, `VALUE`를 `final`로 지정하여 상수로 만드는 경우는 ClassLoading이 발생하지 않는다.

---

즉, static 필드와 메소드를 가지지 않는 Eager Initialization 방식의 Singleton의 경우 `Class.forName()`과 같은 메소드를 호출하지 않는 한, Eager Initialization 방식을 써도 `INSTANCE`에 접근하지 않는 한 객체 초기화가 발생하지 않는다.

## 정적 팩터리 메서드 사용(Lazy Initialization)

Singleton의 static 필드나 메소드를 사용해야 하지만, 싱글톤 객체는 초기화시키지 않고 싶은 경우가 있을 수 있다.

이러한 경우에는 Lazy Initialization 방식을 사용할 수 있다. Lazy Initialization 방식은 Singleton 객체가 필요한 경우에만 이를 초기화(초기화되어 있지 않은 경우에만)하고, Singleton 객체를 반환받을 수 있는 진입점(static 메소드)를 제공하는 방식이다.

**Singleton.java**
{% highlight java %}
public class Singleton {
  public static Singleton INSTANCE;
  
  private Singleton() {
    System.out.println("Simple singleton instance initialized.");
  }
  
  public static Singleton getInstance(){
    if(INSTANCE == null)
      INSTANCE = new Singleton();
    return INSTANCE;
  }
}
{% endhighlight %}

**Test.java**
{% highlight java %}
public class Test {
  public static void main(String[] args) throws Exception {
    System.out.println("Hello world");
    Singleton singleton = Singleton.getInstance();
  }
}
{% endhighlight %}

명시적으로 `getInstance` 메소드를 호출하지 않는 이상 Singleton 객체(`INSTANCE`)가 초기화되지 않기 때문에, Eager Initialization 방식의 단점을 보완할 수 있다.

그러나 Lazy Initialization 방식은 Multi Thread 환경에서의 문제점을 가지고 있다.

## Multi Thread 환경에서의 Lazy Initialization 

Multi Thread 환경에서 서로 다른 Thread들이 동시에 `getInstance`를 호출한다고 생각해보자. 위의 `getInstance`는 Multi Thread 환경에 대한 고려가 되어 있지 않기 떄문에, `if(INSTANCE == null)`이 여러번 참인 경우가 발생하여 여러 개의 객체가 생성될 수 있다.

{% highlight java %}
public class Singleton {
  public static final AtomicInteger NUM_INSTANCE = new AtomicInteger(0);
  private static Singleton INSTANCE;

  private Singleton() {
    System.out.println("Constructor called");
  }

  public static Singleton getInstance() {
    if (INSTANCE == null) {
      INSTANCE = new Singleton();
      NUM_INSTANCE.incrementAndGet();
    }
    return INSTANCE;
  }

  public static int getNumberOfInstance() {
    return NUM_INSTANCE.get();
  }

  public static void main(String[] args) throws Exception {
    Thread[] threads = new Thread[1000];
    for (int i = 0; i < threads.length; i++) {
      threads[i] = new Thread(new Runnable() {
        public void run() {
          Singleton.getInstance();
        }
      });
    }
    for (int i = 0; i < threads.length; i++) {
      threads[i].start();
    }


    for (int i = 0; i < threads.length; i++) {
      threads[i].join();
    }

    System.out.println(Singleton.getNumberOfInstance());
  }
}
{% endhighlight %}

Multi Thread 환경에서 Singleton 객체를 Lazy Initialization하는 몇가지 방법을 소개한다.

### 정적 팩터리 메서드에 동기화 키워드 사용하기

Singleton 객체를 생성하는 정적 팩터리 메서드 자체에 `synchronized`를 붙여주거나, 내부 초기화 구문을 `synchronized` 구문으로 감싸주는 방법이 있다.

**정적 팩터리 메서드 자체에 `synchronized` 붙이기**

기존의 `public static Singleton getInstance()` 를 `public static synchronized Singleton getInstance()`와 같이 바꾸는 방법이다.

**내부 초기화 구문을 `synchronized`로 감싸주기**

초기화에 사용할 Lock 객체를 만들고, Singleton 객체의 초기화 여부 확인 시 동기화하는 방법이다.

{% highlight java %}
private static Object lock = new Object();

public static Singleton getInstance() {
  synchronized (lock) {
    if (INSTANCE == null) {
      INSTANCE = new Singleton();
    }
  }
  return INSTANCE;
}
{% endhighlight %}

---

동기화 방식은 Multi Thread 환경에서 객체를 안전하게 1개를 생성할 수 있지만, 객체 획득 시의 병렬성을 잃어버린다는 단점이 존재한다. 또한 초기 호출 이후에는 객체 초기화 확인 구문(`if(INSTANCE) == null)`)이 계속해서 참을 반환하기 때문에 불필요한 Lock을 거는 상황이 발생하게 된다.

### DCL(Double-Checked-Locking) 방식 사용하기

Multi Thread 환경에서 병렬성을 확보하기 위해서는 동기화 구문을 필요한 곳에서만 사용해야 한다. DCL 방식은 동기화를 최소화하므로써 위의 방식의 단점(불필요한 Lock을 거는)을 없앤 방식이다.

{% highlight java %}
private static Object lock = new Object();

public static Singleton getInstance() {
  if (INSTANCE == null) {
    synchronized (lock) {
      if (INSTANCE == null) {
        INSTANCE = new Singleton();
      }
    }
  }
  return INSTANCE;
}
{% endhighlight %}

첫번째 if문을 통해 객체가 초기화되었는지 검사할 때는 동기화를 하지 않고, 두번째 if문을 통해 객체가 초기화되었는지 검사할 때는 동기화를 수행한다. `getInstance` 호출 후 충분한 시간이 지나 동기화가 필요 없을 때는 첫번째 if문만 실행하므로써 동기화에 따른 병렬성 감소를 감소시킬 수 있는 방식이다.

언뜻 보기에는 동기화 문제도 없고 성능적으로 뛰어난 방식이라고 생각할 수 있다. 그러나 위 코드는 문제점을 가지고 있다.

#### 위 DCL 코드의 문제점과 해결

위 DCL 코드는 JVM의 Low-Level 동작 방식때문에 문제가 발생한다. 

예를 들어 2개의 Thread(A, B)가 `getInstance`를 호출했다고 생각해보자. Thread A는 `INSTANCE = new Singleton()` 구문을 수행하고 있고, 이 때 Thread B는 첫번째 if문에서 INSTANCE의 초기화 여부를 검사(`if(INSTANCE == null)`)하고 있다. 여기서 Thread A가 수행하는 '객체 생성'과 '변수 할당' 구문은 한번에 실행되지 않고, 특정 시점에는 INSTANCE에는 일부만 초기화된 객체가 할당되어 있을 수 있다. 이 때 Thread B는 non-null check만 하기 때문에 일부만 초기화된 객체를 가져다 쓰게 되어 Crash가 발생할 수 있다.

이러한 문제점을 해결하기 위해서는 INSTANCE 객체에 `volatile` 키워드를 붙여주는 것이다. `volatile`은 원자적으로 동작하기 때문에 첫번째 if문이 완전히 초기화된 객체를 얻거나 아닌 경우 둘 중 하나로 동작하게 된다.

### Lazy Holder 방식 사용하기

Multi Thread 환경에서도 잘 동작하고, Lazy Initialization도 가능한 방식이다.

{% highlight java %}
public class LazySingletonHolder {
  public static Singleton getInstance() {
    return Singleton.INSTANCE;
  }

  public static class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {

    }
  }
}
{% endhighlight %}

위 코드를 보면 'Lazy Initialization 방식이 아니지 않은가?' 라고 생각할 수 있다. `LazySingletonHolder`의 `getInstance`에서 `Singleton` 클래스의 `INSTANCE`를 그대로 반환하기 때문이다.

그러나 `Singleton` 클래스의 `INSTANCE`는 `Singleton` 클래스의 클래스 로딩이 일어날 때 발생하게 되고, 이는 `LazySingletonHolder`의 `getInstance`가 호출되었을 때이다.

또한 `INSTANCE`는 `static final` 이기 때문에 클래스 로딩에 의해 1회만 초기화되고, 이로써 Multi Thread 환경에서의 문제점도 없어지게 된다.
