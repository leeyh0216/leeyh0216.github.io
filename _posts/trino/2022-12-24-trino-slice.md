---
layout: post
title:  "Java Object의 Memory Layout과 Trino의 Slice"
date:   2022-12-24 15:00:00 +0900
author: leeyh0216
tags:
- java
- trino
- airlift
- unsafe
---

# Java Object의 Memory Layout

## Data structure alignment

### WORD

CPU에서 어떤 작업을 하기 위해서는 메모리 영역의 데이터를 레지스터로 옮겨야 한다. 이 때 CPU가 레지스터로 데이터를 옮겨오는 단위를 [WORD](https://ko.wikipedia.org/wiki/%EC%9B%8C%EB%93%9C_(%EC%BB%B4%ED%93%A8%ED%8C%85))라고 한다. WORD의 크기는 CPU마다 다르다.

* 32bit CPU에서는 WORD의 크기가 32bit
* 64bit CPU에서는 WORD의 크기가 64bit

다만 Intel x86에서 WORD=16bit로 정해놓은 MACRO가 존재하였기 때문에, 어셈블리 상에서는 아직도 WORD=16bit 로 표현한다. DOUBLE WORD=32bit, QUAD WORD=64bit로 표현한다.

```
int main(void) {
    int arr[10] = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100};
    for(int i = 0; i < 10; i++) {
        arr[i] = i;
    }

    return 0;
}
```

위의 코드를 어셈블리로 변경했을 때, Loop 내의 `arr[i] = i` 구문은 아래와 같이 표현된다.

```
mov     eax, DWORD PTR [rbp-4]
cdqe
mov     edx, DWORD PTR [rbp-4]
mov     DWORD PTR [rbp-48+rax*4], edx
```

여기서 `eax`, `edx` 등의 레지스터로 DWORD PTR 씩 옮기는 것을 볼 수 있는데, 32bit 메모리 주소만큼의 데이터를 복사하는 것으로 이해할 수 있다.

### Aligned Memory Access vs Misaligned Memory Access

CPU는 메모리 공간에 접근할 때, WORD 단위로 접근한다. 만일 32bit CPU가 Memory를 접근할 때는 (0x0 ~ 0x3), (0x4 ~ 0x7) ... 와 같이 4의 배수씩 끊어서 메모리 공간에 접근한다.

4byte 짜리 데이터가 (0x0 ~ 0x3) 주소에 위치해있다고 생각해보자. CPU에서 레지스터에 해당 데이터를 올려놓기 위해서는 (0x0 ~ 0x3) 주소 구간 1번만을 접근하면 된다. 이러한 메모리 접근을 Aligned Memory Access라고 한다.

반면 같은 데이터가 0x1 ~ 0x4 공간에 위치해있다면, CPU는 (0x0 ~ 0x3), (0x4 ~ 0x7)과 같이 2회 메모리에 접근해야 온전한 데이터를 가져올 수 있다. 이러한 메모리 접근을 Misaligned Memory Access라고 한다.

### Data structure padding

```
#include<stdio.h>

struct MyStruct {
        short a;
        int b;
};
int main(void) {
        printf("%d", sizeof(MyStruct));
        return 0;
}
```

우리가 알고 있는 프로그래밍 지식으로 생각하면 위 코드는 6을 출력해야 한다. `short`형 2byte + `int`형 4byte = 6byte이기 때문이다. 그러나 실제로는 8이라는 결과가 출력된다. 이는 CPU의 메모리 접근을 최적화시키기 위하여 컴파일러에서 Padding이라는 것을 삽입하기 때문이다.

Padding 없이 위 구조체를 메모리 공간에 할당한다면,

* 0x0 ~ 0x1: `a`
* 0x2 ~ 0x3: `b`의 상위 2byte
* 0x4 ~ 0x5: `b`의 하위 2byte

와 같이 할당된다. CPU에서 `a`에 접근하기 위해서는 1개 WORD만 가져오면 되지만, `b`에 접근하기 위해서는 2개 WORD를 가져와야 한다.

이러한 문제를 해결하기 위해 Compiler는 Padding을 삽입하여 구조체를 위한 메모리 공간을 아래와 같이 마련한다.

* 0x0 ~ 0x1: `a`
* 0x2 ~ 0x3: Padding
* 0x4 ~ 0x7: `b`

이렇게 되면 메모리 공간은 손해이지만, CPU의 메모리 접근은 최적화된다.

> 추가로 `aligned` 키워드를 통한 Cache Line 최적화 내용도 Data Memory Alignment에 영향을 미치긴 하는데, 현재의 내용과 연관이 깊지 않아 다음에 시간이 되면 다루려 한다.

## Java Ordinary Object Pointer와 Memory Layout

Java에서는 객체를 가리키기 위한 포인터를 Ordinary Object Pointer(OOP)라 한다.

* [`instanceOop`](https://github.com/openjdk/jdk15/blob/master/src/hotspot/share/oops/instanceOop.hpp): 단일 객체를 표현하기 위한 OOP
* [`arrayOop`](https://github.com/openjdk/jdk15/blob/master/src/hotspot/share/oops/arrayOop.hpp): 배열을 가리키기 위한 OOP

이 OOP들은 모두 [`oopDesc`](https://github.com/openjdk/jdk15/blob/master/src/hotspot/share/oops/oop.hpp)라는 클래스를 기반으로 한다. `oopDesc`는 `mark word`와 `klass word` 필드를 포함하고 있다.

* `mark word`: Object Header 정보를 포함하고 있다. 32bit 운영체제에서는 4byte, 64bit 운영체제에서는 8byte 크기를 가진다.
* `klass word`: Language Level에서의 메타데이터를 포함하고 있다. 4byte 크기를 가진다.

즉, 모든 자바 객체들은 기본적으로 12byte의 Overhead(64bit 운영체제 기준, 8byte `mark word` + 4byte `klass word`)를 가지고 있다.

### Short 자료형 Memory Layout 확인해보기

Java Object Layout 라이브러리를 사용하면, 자바 객체의 메모리 레이아웃을 확인할 수 있다. 아래 코드를 통해 `Short` 타입의 메모리 구조에 대해 확인할 수 있다.

```
short a = 1;
System.out.println(ClassLayout.parseClass(Short.class).toPrintable(a));

-- 출력

java.lang.Short object internals:
OFF  SZ    TYPE DESCRIPTION               VALUE
  0   8         (object header: mark)     0x0000004be6566201 (hash: 0x4be65662; age: 0)
  8   4         (object header: class)    0x0003e7b0
 12   2   short Short.value               1
 14   2         (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total
```

* Offset 0 ~ 7: `mark word` 영역. 64bit 운영체제이기 때문에 8byte
* Offset 8 ~ 11: `klass word` 영역. 4byte
* Offset 12 ~ 13: 실제 Short 데이터 영역. 2byte
* Offset 14 ~ 15: Padding

위에서 설명했듯 CPU에서는 메모리 접근을 최적화하기 위해 WORD 단위로 접근한다. 64bit 운영체제에서는 WORD가 8byte(64bit)이기 때문에 JVM에서는 이에 맞추어 Memory Alignment 단위를 8byte로 지정하고 있다.

```
System.out.println(VM.current().details());

-- 출력

# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# WARNING | Compressed references base/shifts are guessed by the experiment!
# WARNING | Therefore, computed addresses are just guesses, and ARE NOT RELIABLE.
# WARNING | Make sure to attach Serviceability Agent to get the reliable addresses.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
```

위의 "Objects are 8bytes aligned."라는 메시지를 통해 Java의 Memory Alignment 단위가 8byte라는 것을 확인할 수 있다. 

```
java.lang.Short object internals:
OFF  SZ    TYPE DESCRIPTION               VALUE
  0   8         (object header: mark)     0x0000004be6566201 (hash: 0x4be65662; age: 0)
  8   4         (object header: class)    0x0003e7b0
 12   2   short Short.value               1
 14   2         (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total
```

다시 Short 자료형에 대해 생각해보자면, 12byte의 header(`mark`, `klass`)와 2byte(Short)의 데이터를 합쳐 총 14byte의 크기임을 확인할 수 있다. 그러나 WORD 단위로 데이터를 Align하기 위해 마지막에 2byte Padding을 추가하는 것이다. Padding을 추가하였기 때문에 2byte의 Space Loss(이 경우 객체 External Fragmentation이기 때문에 External Loss가 추가)가 발생하는 것을 확인할 수 있다.

### short 배열 Memory Layout 확인해보기

```
short[] arr = new short[1025];
System.out.println(ClassLayout.parseClass(short[].class).toPrintable(arr));

-- 출력
[S object internals:
OFF  SZ    TYPE DESCRIPTION               VALUE
  0   8         (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4         (object header: class)    0x00006d08
 12   4         (array length)            1025
 12   4         (alignment/padding gap)   
 16   0   short [S.<elements>             N/A
 16   0         (object alignment gap)    
Instance size: 2072 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

배열에 대한 OOP에는 배열 길이에 대한 정보를 포함하고 있다. `mark`, `klass` 헤더 정보 + 배열 길이 + 데이터로 구성된다.

* Offset 0 ~ 7: `mark word` 영역. 64bit 운영체제이기 때문에 8byte
* Offset 8 ~ 11: `klass word` 영역. 4byte
* Offset 12 ~ 15: 배열의 길이. 4byte

OOP 공간은 16byte이기 때문에 별도의 Padding이 추가되지 않는다. 16번째 Offset부터 실제 배열의 데이터가 할당되는데, 길이 1025의 배열이므로 총 2050byte를 차지하게 된다. 여기서 알 수 있는 점은 다음과 같다.

* 예상되는 배열 객체의 크기는 2066byte(= 16byte + 2050byte(2byte * 1025))이다.
* 실제 배열 객체의 크기는 2072byte(= 16byte + 2050byte(2byte * 1025) + 6byte(=Padding))이다.

# Java의 Unsafe를 이용한 저수준 데이터 조작

C, C++과 같은 저수준 언어에서는 Pointer를 통한 저수준 데이터 조작이 가능하다.

```
#include<stdio.h>

int main(void){
        int arr[4] = {1,2,3,4};
        int* ptr = arr;
        for(int i = 0; i < 4; i++) {
                printf("%p\n", ptr);

                *ptr = (*ptr) * 10;
                ptr = ptr + 1;
        }
        for(int i = 0; i < 4; i++) {
                printf("%d\n", arr[i]);
        }
        return 0;
}
```

자바에서는 객체의 주소값을 직접 참조하는 기능을 표면적으로는 제공하지 않지만, `Unsafe`를 사용하면 위와 같은 코드를 작성할 수 있다.

```
import sun.misc.Unsafe;

import java.lang.reflect.Field;

import static sun.misc.Unsafe.ARRAY_BYTE_BASE_OFFSET;
import static sun.misc.Unsafe.ARRAY_INT_INDEX_SCALE;

public class UnsafeIteration {
    public static Unsafe getUnsafe() throws Exception {
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        return (Unsafe) theUnsafe.get(null);
    }

    public void mulAndPrint() throws Exception {
        Unsafe unsafe = getUnsafe();
        int[] arr = {1, 2, 3, 4};

        Object base = arr;
        long address = ARRAY_INT_BASE_OFFSET;

        for(int i = 0; i < 4; i++) {
            System.out.println("Address " + i +"th element: " + address);

            unsafe.putInt(base, address, unsafe.getInt(base, address) * 10);
            address += ARRAY_INT_INDEX_SCALE;

        }
        for(int i = 0; i < 4; i++)
            System.out.println(arr[i]);
    }

    public static void main(String[] args) throws Exception {
        new UnsafeIteration().mulAndPrint();
    }
}
```

위 코드의 `mulAndPrint` 동작은 다음과 같다.

* 특정 객체를 가리키기 위한 Pointer인 `base` Object 객체를 만들고, `base`가 `arr`을 가리키도록 만든다.
* 데이터의 주소를 가리키기 위한 변수인 `address`를 선언한다.
  * Unsafe에서 주소 참조는 두가지로 나뉜다.
    * 절대 주소를 사용하는 방법
    * 상대 주소를 사용하는 방법: 기준 주소(객체 포인터)와 Offset을 사용하여 절대 주소 계산
  * 여기서는 기준 주소와 Offset을 사용하는 방법을 사용한다.
  * 위에서 설명했듯, 배열 객체는 16byte의 Header(`mark` + `klass` + 배열 길이)를 사용한다. 그렇기 때문에 `ARRAY_INT_BASE_OFFSET`을 사용하여 시작 주소를 옮긴다.
* Loop를 순회한다.
  * `unsafe.getInt(base, address)`: 기준 주소에 Offset 주소를 더한 뒤, 해당 주소값으로부터 4byte int를 읽는다.
  * `unsafe.putInt(base, address, unsafe.getint(base, address) * 10)`: 기준 주소에 Offset 주소를 더한 뒤, 해당 주소값에 4byte int 데이터를 넣는다.
  * `address += ARRAY_INT_INDEX_SCALE`: 주소 값에 4byte만큼을 증가시킨다.

> 위 연산들은 모두 `ByteBuffer`를 통해서도 동일하게 수행할 수 있다. [How to speed up a byte[] lookup to be faster using sun.misc.Unsafe?](https://stackoverflow.com/questions/12226123/busted-how-to-speed-up-a-byte-lookup-to-be-faster-using-sun-misc-unsafe) 글을 참고해보면 Unsafe 방식이 10 ~ 15% 가량의 성능 이점이 있을 수 있다고 되어 있다.
>
> `Unsafe`는 잘못 사용할 경우 저수준 언어에서의 Segmentation Fault 등의 문제를 동일하게 겪을 수 있기 때문에, Java에서는 Public API로 노출되어 있지 않다.
>
> 다만 Trino나 Spark과 같은 성능이 중시되는 애플리케이션에서는 MicroOptimization 또한 중요하기 때문에 Unsafe를 사용한다.

# Trino의 Slice

사실 정확히 표현하자면 Trino에서 사용하는 Airlift의 [Slice](https://github.com/airlift/slice) 라이브러리이다. 

Slice는 "Slice is a Java library for efficiently working with heap and off-heap memory."와 같은 소개를 하고 있는데, Trino에서는 데이터의 저장/전송 등에 Slice를 사용한다.

내부적으로 사용되는 라이브러리이기 때문에, 별도의 공식 문서가 없어 코드 자체를 확인해보았다.

## Slice

`Unsafe`를 사용하여 저수준 데이터 조작을 수행하는 클래스이다. [Slice](https://github.com/airlift/slice/blob/master/src/main/java/io/airlift/slice/Slice.java) 에서 코드 확인이 가능하다.

6개의 멤버 변수를 포함하고 있다.

```
private final Object base;

private final long address;

private final int size;

private final long retainedSize;

private final Object reference;

private int hash;
```

`reference`와 `hash`를 제외한 각 변수의 역할은 다음과 같다.

* `base`: Slice 객체가 가리키는 원본 객체이다. `Slice`에서 `Unsafe`의 상대 주소를 사용하기 때문에, 기준점이 되는 주소를 제공한다고 볼 수 있다.
* `address`: 원본 객체의 데이터 시작 주소를 의미한다. 위에서 설명했듯 헤더 데이터 등을 제외하고 실제로 데이터가 시작되는 주소 값을 의미한다.
* `size`: 데이터의 크기이다.
* `retainedSize`: Slice 객체의 크기를 포함한 데이터의 크기이다.

### 생성자

`Slice`의 생성자는 모두 `private` 접근 제한자를 가지고 있다. 외부에서의 생성은 팩토리 역할을 수행하는 `Slices` 클래스에 존재한다.

간단하게 `int` 타입 배열을 다루는 `Slice`의 생성자에 대해 확인해본다.

```
Slice(int[] base, int offset, int length)
{
    requireNonNull(base, "base is null");
    checkPositionIndexes(offset, offset + length, base.length);

    this.base = base;
    this.address = sizeOfIntArray(offset);
    this.size = multiplyExact(length, ARRAY_INT_INDEX_SCALE);
    this.retainedSize = INSTANCE_SIZE + sizeOf(base);
    this.reference = (offset == 0 && length == base.length) ? COMPACT : NOT_COMPACT;
}
```

* `base`는 `int` 배열을 가리킨다.
* `address`는 `sizeOfIntArray`를 호출하여 초기화한다.
  ```
  public static long sizeOfIntArray(int length)
  {
    return ARRAY_INT_BASE_OFFSET + (((long) ARRAY_INT_INDEX_SCALE) * length);
  }
  ```
  * Slice 생성 시 원본 배열의 일부 구간(offset ~ offset + length - 1)을 기준으로 생성할 수 있기 때문에, 데이터의 시작 주소(`address`)는 `ARRAY_INT_BASE_OFFSET`에 Skip할 `offset` * `ARRAY_INT_INDEX_SCALE`만큼을 더해주어야 한다.
* `size`는 데이터의 실제 크기를 계산하여 집어넣는다.
* `retainedSize`는 `Slice` 객체 자체의 크기(`INSTANCE_SIZE`)에 데이터의 크기를 더한 크기이다.
  ```
  private static final int INSTANCE_SIZE = toIntExact(ClassLayout.parseClass(Slice.class).instanceSize());
  ```

### `set`, `get` 계열 메서드

`Slice`의 `set` 계열 메서드를 통해 `Slice`를 구성하는 데이터의 특정 Offset에 데이터를 설정할 수 있다.

```
public void setInt(int index, int value)
{
    checkIndexLength(index, SIZE_OF_INT);
    setIntUnchecked(index, value);
}

void setIntUnchecked(int index, int value)
{
    unsafe.putInt(base, address + index, value);
}
```

`setInt`를 통해 특정 Index(`index`)에 값(`value`)를 설정할 수 있다. `setIntUnchecked`는 `Unsafe`의 `putInt`를 통해 넣어야 할 주소(`base` + `address` + `index`)에 값을 설정하는 것을 확인할 수 있다.

반대로 `get` 계열 메서드를 통해 `Slice`를 구성하는 데이터의 특정 Offset의 데이터를 조회할 수 있다.

```
public int getInt(int index)
{
    checkIndexLength(index, SIZE_OF_INT);
    return getIntUnchecked(index);
}

int getIntUnchecked(int index)
{
    return unsafe.getInt(base, address + index);
}
```

## Slices

`Slices`는 `Slice` 생성을 위한 정적 팩토리 메서드를 제공하는 클래스이다. 아래는 `int`형 배열을 기반으로 하는 `Slice`를 생성하는 `wrappedIntArray` 코드이다.

```
public static Slice wrappedIntArray(int[] array, int offset, int length)
{
    if (length == 0) {
        return EMPTY_SLICE;
    }
    return new Slice(array, offset, length);
}
```

## Usage Slice in Trino

`Slice`는 Trino의 데이터 조작에서 굉장히 많이 사용되고 있으며, 특히 파일 입/출력 시 버퍼로 자주 사용된다. `OrcOutputBuffer` 등에서 사용되니 코드를 하나씩 확인해보면 좋을 것 같다.

# 참고자료

* [Data Structure Alignment - Wikipedia](https://en.wikipedia.org/wiki/Data_structure_alignment)
* [Java Memory Layout - Baeldung](https://www.baeldung.com/java-memory-layout)