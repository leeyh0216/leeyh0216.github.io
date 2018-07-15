---
layout: post
title:  "[Scala] Scala의 Type 소거"
date:   2017-04-23 23:00:00 +0900
author: leeyh0216
categories: dev lang scala
---

> 이 문서는 Scala Type 관리에 대해 서술한 문서입니다.

### JVM의 Type 소거(Type Erase)
Java에서는 Generic을 지원한다. 하지만 Generic은 컴파일 단계에서는 Object Type을 강제할 수 있지만, 컴파일 단계에서 Type 정보가 사라지기 때문에(Type Erase) Runtime 때에는 Type을 강제할 수 없다.

예를 들어 다음과 같은 Generic Class가 있다고 가정하자.
{% highlight java %}
package com.leeyh0216.test;

import java.util.ArrayList;
import java.util.List;

/**
 * Created by leeyh0216 on 17. 4. 24.
**/
public class TypeErase<T> {
    private T item;

    public void setItem(T item){
        this.item = item;
    }

    public T getItem(){
        return this.item;
    }

    public String getClassName(){
        return item.getClass().getName();
    }
}

{% endhighlight %}

위 코드는 컴파일 시 다음과 같이 변경된다.
{% highlight java %}
package com.leeyh0216.test;

import java.util.ArrayList;
import java.util.List;

/**
 * Created by leeyh0216 on 17. 4. 24.
**/
public class TypeErase<Object> {
    private Object item;

    public void setItem(Object item){
        this.item = item;
    }

    public Object getItem(){
        return this.item;
    }

    public String getClassName(){
        return item.getClass().getName();
    }
}

{% endhighlight %}

테스트를 위해 다음과 같은 코드를 작성하여 실행시켜보면, String Generic으로 선언한 TypeErase 클래스의 item 필드의 Type이 Object로 변경되는 것을 확인할 수 있다.
{% highlight java %}
package com.leeyh0216.test;

import org.junit.After;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
/**
 * Created by leeyh0216 on 17. 4. 24.
**/
public class TypeEraseTest {
    @Before
    public void setUp() throws Exception{

    }

    @After
    public void tearDown() throws Exception{

    }

    @Test
    public void testTypeErase(){
        TypeErase<String> typeErase = new TypeErase<String>();
        typeErase.setItem("hello world");
        Assert.assertEquals("Object",typeErase.getClassName());
    }
}
{% endhighlight %}

이는 Generic이 Java5에서 생겼기 때문에 하위 호환을 위하여 Generic에 사용된 Type 정보를 Object로 변경하는 것이다.
때문에, Generic이 아닌 다음과 같은 구문도 사용할 수 있게 된다.
{% highlight java %}
//Integer Type을 지정하지 않았더라도 사용할 수 있다.
List list = Arrays.asList(1,2,3,4,5);
{% endhighlight %}

### Scala의 타입 소거와 해결책
Scala도 JVM 언어이기 때문에, 타입 소거에서 자유로울 수 없다.
이를 보완하기 위하여 Scala에서는 TypeTag, Manifest, ClassTag 등을 지원한다.
이 모듈들은 컴파일 이후(런타임)에도 Type 정보를 참조할 수 있도록 Scala에서 지원하는 요소들이다.

[TypeTag, Manifest - Scala](http://docs.scala-lang.org/overviews/reflection/typetags-manifests.html)

추후 계속 작성...


