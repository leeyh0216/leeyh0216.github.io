---
layout: post
title:  "[Javascript] 자바스크립트 기초(변수, 상수, 객체)"
date:   2018-03-31 15:24:00 +0900
author: leeyh0216
categories: javascript essential
---

# 자바스크립트 기초

> Java, Scala와 같은 JVM 기반 언어만 배우다가, 자바스크립트를 이용하여 프론트 엔드 개발할 일이 생겨 해당 문서를 작성하였습니다. 이전에 자바스크립트를 사용해본 적이 있고, 어느정도 언어에 대한 이해도가 있는 상태에서 작성되었기 때문에, 초보자가 읽기에는 부적합할 수 있습니다.

## 변수와 상수

변수는 이름 있는 값을 의미하고, 변수에 할당된 값은 언제든 변경할 수 있다.
아래는 a라는 변수를 선언하고, 선언과 동시에 "hello world"라는 문자열을 할당한 뒤 출력하는 예제이다.

~~~
let a = "hello world"
console.log(a)
~~~
> **ES6 이전에는 var 키워드를 사용**하여 변수를 만들었었는데, **ES6부터는 let이라는 키워드를 사용**하고 있다.

선언한 변수를 아래와 같이 변경할 수도 있다. 자바스크립트는 동적 언어이기 때문에 변수의 타입이 엄격하게 지정되어 있지 않아, 다른 타입의 변수를 지정해 줄 수도 있다.

~~~
let a = "hello world"
console.log(a)
a = 1.1 //변수에 다른 값을 할당할 수 있다. 타입 또한 다른 타입으로 변경할 수 있다.
console.log(a)
~~~

단, 한번 let을 이용하여 변수를 선언한 이후에 동일한 변수를 다시 선언한다면 오류가 발생한다.

~~~
let a = "hello world"
console.log(a)
let a = 1.1 //오류 발생
~~~

일반적으로 Java나 Scala같은 언어에서는 변수를 선언하고 아무 값도 지정해주지 않을 경우 Primitive Type의 경우 Initial value(int의 경우 0, boolean의 경우 fasle 등)가 할당되지만, Javascript의 경우 undefined가 할당된다.

~~~
let a;
console.log(a)
~~~

상수는 const 라는 키워드를 이용하여 정의할 수 있다. ES6 이전에는 const도 없었던 것 같은데, ES6 부터는 const라는 값을 이용하여 상수를 정의할 수 있다.

~~~
const NAVER = "http://naver.com"
console.log(NAVER)
~~~

## 타입

Javascript에는 primitive와 object가 존재한다.
primitive은 아래와 같은 타입을 지원한다.

* Number
* String
* Boolean
* Null
* Undefined
* Symbol

다른 언어와는 달리 숫자 타입이 다양하지 않고 Number만 존재하며, Number 또한 정확도가 아주 높지 않다고 한다.

object의 경우 아래와 같은 타입이 존재한다.

* Array
* Date
* RegExp
* Map, WeakMap
* Set, WeakSet

### null과 undefined

일반적으로 JVM 언어에서 변수를 선언하고 값을 할당하지 않았을 경우 null 값이 변수에 들어있다. Javascript의 경우 변수를 선언하고 값을 할당하지 않았을 경우 undefined가 들어있다. Javascript에서의 undefined는 개발자가 쓸 수도 있지만, 써서는 안되는 값이다.

즉, 언어적인 측면에서 값을 할당하지 않았다는 것을 말해주는 Placeholder 값이며, 개발자가 임의로 해당 값은 아무것도 없다는걸 표현해주고 싶을 경우에는 null을 사용해야 한다.

## Object

자바스크립트의 Object는 JVM 언어의 Object와는 약간 개념이 다르다.
JVM 언어에서는 보통 Class를 선언하고, new 키워드를 이용하여 클래스의 객체를 생성한다. 하지만 Javascript에서의 Object는 정적으로 선언된 Class를 인스턴스화 시킨다는 개념이 아닌, 동적으로 값이나 함수를 할당할 수 있는 컨테이너의 개념이다.

~~~
let a = {}
a.name = "leeyh0216" //a라는 object에 name 필드를 만들고, "leeyh0216"이라는 값을 할당한다.
a.printName = function(){
    console.log(this.name)
} //a라는 object에 printName이라는 함수를 만든다.
//선언한 값과 함수를 호출해본다.
console.log(a.name)
a.printName()

//또 다른 값과 함수를 할당해본다.
a.age = 27
a.printAge = function(){
    console.log(this.age)
}
//할당한 값과 함수를 출력, 실행해본다.
console.log(a.age)
a.printAge()
~~~

선언한 필드나 함수는 배열처럼 []로도 접근이 가능하다.

~~~
let a = {}
a.name = "leeyh0216"
a.printName = function(){
    console.log(this.name)
}

console.log(a["name"])
a["printName"]()
~~~

선언할 때 값과 함수를 설정해줄 수도 있다.

~~~
let a = {
    name: "leeyh0216",
    printName: function(){
        console.log(this.name)
    }
}

console.log(a.name)
a.printName()
~~~

