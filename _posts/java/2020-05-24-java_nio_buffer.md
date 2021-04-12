---
layout: post
title:  "Java NIO - 2. Buffers"
date:   2020-05-24 22:30:00 +0900
author: leeyh0216
tags:
- java
---

* [Java NIO - 1. 왜 자바의 IO 패키지는 느린가?](https://leeyh0216.github.io/2020-05-10/java_nio_why_java_io_slow)
* **Java NIO - 2. Buffers**

# 개요

[Java NIO - 1. 왜 자바의 IO 패키지는 느린가?](https://leeyh0216.github.io/2020-05-10/java_nio_why_java_io_slow)에 이어 Java NIO에서 도입된 Buffer에 대해 알아보고자 한다.

# Buffers

**`Buffer` 객체는 고정 크기의 데이터를 담는 컨테이너**이다. 

각 데이터 타입(Primitive data types)에는 이와 대응하는 `Buffer` 클래스가 존재한다. 이 `Buffer`들은 외부적으로는 각 데이터 타입에 대응하는 것으로 보이지만, **내부적으로는 `byte` 타입에 종속**되어 있다. 즉, 저장된 데이터들을 `byte`로 변환할 수도 있고, `byte`로부터 데이터를 추출할 수도 있다.

## Buffer의 기본 속성과 API

`Buffer`는 내부적으로 **자신에게 설정된 데이터 타입의 배열을 관리**한다. 간단히 말해 배열을 캡슐화한 클래스라고 생각하면 되고, 효과적으로 데이터를 조작할 수 있는 기능을 포함하고 있다.

### Capacity

`Buffer`가 포함할 수 있는 데이터의 최대 갯수이다. 이는 `Buffer`가 생성될 때에만 지정할 수 있고, 변경할 수 없다.

`Buffer`를 생성하는 방법은

* 생성자를 호출
* 정적 메서드인 `allocate`(혹은 `allocateDirect`)을 호출
* 정적 메서드인 `wrap`을 호출

하는 방법 등이 있다. 위 방법을 모두 직/간접적으로 `capacity`를 지정하게 되어 있다. 예를 들어 `allocate`를 호출하는 경우 아래와 같이 매개변수로 `capacity` 값을 넣도록 되어 있다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(4);
{% endhighlight %}

또한 이미 존재하는 배열을 통해 `Buffer`를 생성할 수 있는 `wrap`을 호출하는 경우, 매개변수로 전달된 배열의 길이와 동일한 `capacity`를 가지게 되어 있다.

{% highlight java %}
byte[] byteArr = new byte[]{1, 2, 3, 4};
ByteBuffer buffer = ByteBuffer.wrap(byteArr);
{% endhighlight %}

### Limit

`Buffer`에 임의로 설정된 읽거나 쓸 수 없는 첫번째 Offset이다. 최초에는 `capacity`와 동일한 값을 가지지만, 필요에 의해 `limit`을 설정하여 해당 Offset부터 데이터를 읽을 수 없도록 설정할 수 있다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(4);
System.out.println(buffer.capacity() == buffer.limit());
buffer.put(0, (byte) 'a');
buffer.put(1, (byte) 'b');
buffer.put(2, (byte) 'c');
buffer.put(3, (byte) 'd');

buffer.limit(2);
System.out.println(buffer.capacity() != buffer.limit());
buffer.put(0, (byte) 'a');
buffer.put(1, (byte) 'b');
buffer.put(2, (byte) 'c');
{% endhighlight %}

위 예제에서 `allocate(4)`를 통해 생성한 `Buffer`의 Capacity와 Limit의 값은 처음에는 같다. 또한 0 ~ 3번째 값에 모두 데이터를 읽고 쓸 수 있었다.

하지만 `limit(2)`를 호출한 이후에는 Capacity와 Limit이 다른 값을 가지게 되고, Limit으로 지정한 2번째 Offset에 값을 쓰려고 시도하는 경우 `java.lang.IndexOutOfBoundsException` 오류가 발생하는 것을 확인할 수 있다.

### Position

다음으로 읽거나 쓸 값의 Offset을 의미한다. 처음 `Buffer`를 생성했을 때는 0 값을 가지고 있으며, `put`이나 `get` 등의 메서드를 호출할 경우 1씩 증가하게 되어 있다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(4);
System.out.println(buffer.position());
buffer.put((byte)'a');
System.out.println(buffer.position());
buffer.put((byte)'b');
System.out.println(buffer.position());
{% endhighlight %}

위 예제에서 첫 번째 출력되는 Position 값은 0이다. 'a' 값을 쓸 때는 현재 Position인 0번째 Offset에 기록한 후 Position 값이 자동으로 1 증가한다. 'b' 값을 쓸 때도 현재 Position인 1번째 Offset에 기록한 후 Position 값이 자동으로 1 증가하여 2가 된다.

주의해야 할 점은 `get`, `put` 메서드는 상대값(현재 Position을 기준으로 수행), 절대값(주어진 Offset을 기준으로 수행)을 통한 2가지 방식이 있는데, 이 중 **절대값을 이용한 방식을 사용할 때는 Position 값이 업데이트되지 않는다.**

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(4);
System.out.println(buffer.position());
buffer.put(0, (byte)'a');
System.out.println(buffer.position());
buffer.put(1, (byte)'b');
System.out.println(buffer.position());
{% endhighlight %}

위의 예제는 절대값을 이용한 방식을 사용했기 때문에 0, 1번째 Offset에 'a'와 'b'가 기록되기는 했지만 Position은 계속해서 0으로 남아있게 된다.

### Mark

Mark는 특정 위치를 기억해놓는 기능이다. `mark` 메서드를 호출하는 경우 현재 Position이 내부적으로 Marking되고, 추후 `reset` 메서드를 호출하는 경우 Marking되었던 Offset으로 Position이 설정된다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(4);
buffer.put((byte) 'a');
buffer.put((byte) 'b');
buffer.mark();
buffer.put((byte) 'c');
buffer.put((byte) 'd');
buffer.reset();
buffer.put((byte)'e');
buffer.put((byte)'f');

System.out.println(String.format("Offset 2: %c, Offset 3: %c", buffer.get(2), buffer.get(3)));
{% endhighlight %}

위의 예제에서 Position이 2인 상태에서 `mark`를 호출하였다. 이후 2, 3번째 Offset에 'c', 'd'를 넣은 뒤, `reset`을 호출하여 다시 Position을 2로 되돌렸다. 이후 'e', 'f'에 대해 `put`을 호출할 때는 2, 3번째에 값을 넣게 된다.

## Buffer의 추가 API

`Buffer`는 기본 API 이외에도 편리한 사용을 위한 API를 추가로 제공한다.

### Buffer의 Invocation Chaining

`Buffer`의 추가 API에 대해서 알아보기 전에 `Buffer`의 Invocation Chaining에 대해 알아본다.

`Buffer` 객체를 사용할 때 'h','e','l','l','o' 라는 문자를 기록하고, 이 후 'l','l','o'라는 문자들을 'n','r','y'라는 문자들로 변경해야한다고 생각해보자. 아마 Invocation Chaining이라는 속성을 사용하지 않는다면 아래와 같이 구현해야 할 것이다.

{% highlight java %}
buffer.put((byte)'h');
buffer.put((byte)'e');
buffer.mark();
buffer.put((byte)'l');
buffer.put((byte)'l');
buffer.put((byte)'o');
buffer.reset();
buffer.put((byte)'n');
buffer.put((byte)'r');
buffer.put((byte)'y');
{% endhighlight %}

위와 같이 구현하면 라인 수도 많아지고, 연속적이라는 생각이 잘 들지 않게 된다. 이러한 문제점을 해결하기 위해 `Buffer` 클래스는 메서드들의 반환형을 자기 자신(`Buffer`)으로 설정하여, 반환 결과에 대해 다시 메서드를 호출할 수 있게 하였다. 이런 방식을 Invocation Chaining이라고 한다. 주로 Builder 패턴 등에서 자주 활용된다.

{% highlight java %}
buffer.put((byte) 'h').put((byte) 'e')
    .mark().put((byte) 'l').put((byte) 'l').put((byte) 'o')
    .reset().put((byte) 'n').put((byte) 'r').put((byte) 'y');
{% endhighlight %}

### ReadOnly 속성

`Buffer`를 만든 후 이를 수정하지 못하게 할 수 있다. 이는 `Buffer`의 ReadOnly 속성을 사용하면 된다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(5);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');

ByteBuffer readOnlyBuffer = buffer.asReadOnlyBuffer();
System.out.println(String.format("Get 0th offset: %c", readOnlyBuffer.get(0)));
readOnlyBuffer.put(0, (byte) 'a');
{% endhighlight %}

위의 예제와 같이 'h','e','l','l','o'로 이루어진 `Buffer`에 대해 `asReadOnlyBuffer` 메서드를 호출하여 새로운 `readOnlyBuffer` 객체를 생성한 뒤, 이에 대해 `get`과 `put` 메서드를 호출해보았다.

`get`의 경우 정상적으로 수행되는 것을 확인했지만, `put`을 호출할 경우 `java.nio.ReadOnlyBufferException`이 발생하는 것을 확인하였다.

주의해야할 점은 `asReadOnlyBuffer`의 경우 반환형이 `Buffer`라서 Invocation Chaining이라고 생각할 수도 있는데, 이 메서드는 **Caller 객체(`Buffer`)의 내용과 동일한 읽기 전용 `Buffer`를 새로 생성하여 반환한다는 것**이다. 따라서 **기존 `Buffer` 객체의 `put`을 호출하는 경우에는 정상적으로 동작**하는 것을 확인할 수 있다.

public ByteBuffer put(ByteBuffer src) {
    if (src == this) {
      throw createSameBufferException();
    } else if (this.isReadOnly()) {
      throw new ReadOnlyBufferException();
    } else {
      int n = src.remaining();
      if (n > this.remaining()) {
        throw new BufferOverflowException();
      } else {
        for(int i = 0; i < n; ++i) {
          this.put(src.get());
        }

        return this;
      }
    }
  }

위의 `put`의 내부 구현을 보면 `isReadOnly`라는 메서드를 호출하여 현재 `Buffer`의 ReadOnly 속성을 확인하여 ReadOnly로 설정되어 있는 경우 `ReadOnlyBufferException`을 발생시키는 것을 확인할 수 있다.

### Flip

아직 `Channel`에 대해 다루지는 않았지만, `Channel`은 읽기/쓰기의 통로로써 `Channel`을 통해 `Buffer`에 담긴 데이터를 저장/전송하거나 데이터를 `Buffer`에 로드/수신할 수 있다.

`Channel`의 `write` 메서드는 데이터를 쓰는 역할을 수행하는데, 매개변수로 넘긴 `Buffer`에서 데이터를 어떻게 가져갈까? 확인을 위해 `FileChannel`의 내부 코드를 확인하였으며, 실제로 파일에 쓰기를 수행하는 메서드 일부분(`sun.nio.ch.IOUtil` 클래스의 `writeFromNativeBuffer`을 가져와 보았다.

{% highlight java %}
private static int writeFromNativeBuffer(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
  int var5 = var1.position();
  int var6 = var1.limit();

  assert var5 <= var6;

  int var7 = var5 <= var6 ? var6 - var5 : 0;
  boolean var8 = false;
  if (var7 == 0) {
    return 0;
  } else {
    int var9;
    if (var2 != -1L) {
      var9 = var4.pwrite(var0, ((DirectBuffer)var1).address() + (long)var5, var7, var2);
    } else {
      var9 = var4.write(var0, ((DirectBuffer)var1).address() + (long)var5, var7);
    }
    ...
  }
}
{% endhighlight %}

위의 `var1`이 우리가 `FileChannel`의 `write`로 넘긴 `Buffer`(사실 전달된 `Buffer`가 `allocate`로 생성된 `Buffer`일 경우 `allocateDirect`로 생성한 `Buffer`로 변경 후 이를 넘기게 됨)이며, `var5`와 `var6`와 같이 `position`과 `limit`을 사용하는 것을 알 수 있다. 즉, `write` 연산을 위해 `Channel`의 `write`를 호출할 때는 데이터가 위치하는 구간을 `position`과 `limit`으로 지정해야 하는 것이다.

그럼 Capacity가 10인 `Buffer`에 'h','e','l','l','o'가 저장되어 있고, 이를 저장하고 싶다면 아래와 같은 함수를 호출해야 한다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o').limit(buffer.position()).position(0);
{% endhighlight %}

`limit`을 통해 5번째 Offset(마지막으로 기록된 'o'의 다음 Offset)으로 설정하고, `position`을 통해 0번째 Offset(첫번째로 기록한 'h'의 Offset)을 설정하므로써 `Channel`이 Offset 0 ~ 4까지의 데이터를 가져갈 수 있게 해야한다.

이러한 과정을 하나의 메서드로 해결할 수 있는 기능이 `flip`이다. `flip`을 호출하면 현재의 Position을 Limit으로 설정하고, Position을 0으로 변경하게 된다. 아래는 `Buffer`의 `flip` 메소드이다.

{% highlight java %}
public final Buffer flip() {
  limit = position;
  position = 0;
  mark = -1;
  return this;
}
{% endhighlight %}

즉, 위의 예제는 아래와 같이 변경할 수 있다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o').flip();
{% endhighlight %}

`flip`에서 주의해야 할 점은 Mark 또한 초기화되어 버린다는 점과, 2번의 `flip`을 연속적으로 호출할 경우 Limit이 0인 0 크기의 `Buffer`로 바뀌어버린다는 점이다.

### Rewind

Rewind는 Position을 0으로 변경하고 Limit은 변경하지 않는다. 이 기능은 Flip과 함께 사용하면 **다시 쓰기**에 유리하게 사용할 수 있다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o').flip();
System.out.println(String.format("Position: %d, Limit: %d", buffer.position(), buffer.limit()));

FileChannel channel = FileChannel.open(Paths.get("/tmp/channel_test"), StandardOpenOption.APPEND);
channel.write(buffer);
System.out.println(String.format("Position: %d, Limit: %d", buffer.position(), buffer.limit()));

buffer.rewind();
System.out.println(String.format("Position: %d, Limit: %d", buffer.position(), buffer.limit()));
{% endhighlight %}

위 예제를 실행시켜보면 아래와 같은 결과가 출력된다.

```
Position: 0, Limit: 5
Position: 5, Limit: 5
Position: 0, Limit: 5
```

`Channel`의 `write`를 호출하게 되면 Position의 위치가 Limit으로 변경되는데, 이 때 `rewind`를 호출하면 다시 Position이 0이 되므로 `Buffer`의 데이터를 다시 쓸 수 있게 된다.

### Compact

Compact는

* [Position, Limit) 범위에 위치한 데이터를 [0, Limit - Position)범위로 이동하고
* Position을 Limit - Position 위치로 이동하고
* Limit을 Capacity로 변경

하는 방법이다.

두 개의 `Channel`이 있고, `Buffer`(Capacity: 10)를 통해 한 `Channel`(편의상 A 채널)에서 다른 `Channel`(편의상 B 채널)로 데이터를 이동하는 상황이 있다고 가정해보자.

1. A 채널에서 4byte를 읽음. `Buffer`의 Position은 4가 됨.
2. B 채널에 데이터를 쓰기 위해 `flip` 호출. `Buffer`의 Position은 0, Limit은 4가 됨.
3. B 채널에 데이터를 2byte 기록. `Buffer`의 Position은 2, Limit은 4가 됨.

위 상황에서 다시 A 채널의 데이터를 읽은 후 B 채널에 데이터를 기록하는 과정을 반복하기 위해서는 매우 복잡한 처리가 필요하다. 이와 같은 문제를 해결하기 위해 Compact를 사용한다. Compact 사용 시 아래와 같이 개선이 가능하다.

1. A 채널에서 4byte를 읽음. `Buffer`의 Position은 4가 됨.
2. B 채널에 데이터를 쓰기 위해 `flip` 호출. `Buffer`의 Position은 0, Limit은 4가 됨.
3. B 채널에 데이터를 2byte 기록. `Buffer`의 Position은 2, Limit은 4가 됨.
4. `Buffer`의 `compact` 호출. [2, 3] 위치에 있던 데이터가 [0, 1] 위치로 이동하고, Position은 2, Limit은 10이 됨.

위 상태에서 A 채널의 데이터를 읽는 경우 Offset 3부터 정상적으로 읽을 수 있고, 쓰는 경우에도 `flip`만 호출하면 Position 0부터 현재 쓴 위치까지 Limit을 정해 편리하게 쓸 수 있다.

### Duplicate

`Buffer`에서 제공하는 Duplicate 기능을 사용하면 `Buffer`의 전체나 일부분을 복제하여 새로운 `Buffer`를 만들 수 있다.

우선 `Buffer` 전체를 복제하는 방법은 `duplicate`를 사용하는 것이다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');

ByteBuffer duplicated = buffer.duplicate();
System.out.println(String.format("Capacity: %d, Position: %d, Limit: %d", duplicated.capacity(), duplicated.position(), duplicated.limit()));
{% endhighlight %}

위의 예제를 실행한 결과는 아래와 같다.

```
Capacity: 10, Position: 5, Limit: 10
```

`duplicate`을 사용하는 경우 Capacity, Position, Limit 등의 값이 모두 복사되는 것을 알 수 있다.

`Buffer`의 일부를 복제하는 방법은 `slice`를 사용하는 것이다. `slice`는 [Position, Limit) 까지의 영역을 복사하여 새로운 `Buffer`를 만들어낸다.

{% highlight java %}
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte)'h').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o').position(1).limit(3);

ByteBuffer sliced = buffer.slice();
System.out.println(String.format("Capacity: %d, Position: %d, Limit: %d", sliced.capacity(), sliced.position(), sliced.limit()));
System.out.println((char)sliced.get(0));
System.out.println((char)sliced.get(1));
{% endhighlight %}

위의 예제를 실행한 결과는 아래와 같다.

```
Capacity: 2, Position: 0, Limit: 2
```

마지막으로 `asReadOnlyBuffer`를 통한 방법도 있다. 이 방법을 사용하는 경우엔 쓰기가 불가능하다는 것에 유의하자.

# 정리

`Buffer`의 기본적인 내용을 알아 보았다. 배열에 비해 다양한 기능을 제공하여 편하게 데이터를 다룰 수 있다는 장점(특히 `flip`, `rewind` 등의 편의 메소드)이 있지만, `Buffer`가 어떻게 IO 성능을 증가시키는지에 대해서는 나오지 않았다. 이 부분은 다음 글에서 다룰 예정이다.