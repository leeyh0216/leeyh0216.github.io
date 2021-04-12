---
layout: post
title:  "Spring with RabbitMQ(2)"
date:   2018-12-30 10:00:00 +0900
author: leeyh0216
tags:
- spring
- rabbitmq
---

# Spring with RabbitMQ

> RabbitMQ에서 사용하는 `AMQP 0-9-1` 모델과 RabbitMQ의 컨셉을 정리하기 위해 작성하였다.
> Spring과의 자세한 매핑은 이후 글에서 작성할 예정

## AMQP 0-9-1 Model

### High-level Overview of AMQP 0-9-1 and the AMQP Model

#### What is AMQP 0-9-1?

AMQP 0-9-1(Advanced Message Queueing Protocol)은 클라이언트 어플리케이션들이 미들웨어 메시지 브로커를 통해 통신할 수 있도록 하는 메시지 프로토콜을 의미한다.

#### Brokers and Their Role

브로커(Broker)는 Publisher(생산자, 메시지를 브로커에게 전송하는 어플리케이션, Producer라고도 불리운다)에게 메시지를 수신하며, 이를 Consumer(소비자, 메시지를 브로커로부터 수신하여 처리하는 어플리케이션)에게 전달한다.

AMQP-0-9-1은 메시지 프로토콜이기 때문에, Publisher, Broker, Consumer는 서로 다른 장비에서 동작할 수 있다.

#### AMQP 0-9-1 Model in Brief

AMQP-0-9-1 Model에는 위에서 설명한 Producer, Broker, Consumer 말고도 몇가지 개념이 더 존재한다.

* Enchange: Producer로부터 메시지를 수신하여 정해진 규칙(Binding)에 따라 Queue에 전달(복사)한다.
* Binding: Exchange와 Queue를 연결하는 규칙
* Queue: 메시지를 소비하기 전 대기하는 논리적인 장소

메시지가 전달되는 과정은 아래와 같다.

1. Producer는 Exchange에 메시지를 전달한다.
2. Exchange는 메시지의 속성과 일치하는 Binding을 찾는다.
3. 찾은 Binding에 연결된 Queue에 메시지를 복사한다.
4. Consumer는 Pull 혹은 Push 방식으로 Queue에서 메시지를 수신/소비한다.

Network는 완전히 신뢰할 수 없고 Consumer가 메시지를 처리하는 도중 Exception이 발생하여 메시지가 유실될 수 있기 때문에 AMQP Model은 Message Acknowledgement라는 개념이 존재한다.

Broker는 메시지를 Consumer에게 전달했더라도 즉시 Queue에서 메시지를 삭제하지 않고, Consumer로부터 Acknowledge를 수신한 이후에 메시지를 삭제하게 된다.

즉, Consumer가 메시지를 처리하는 도중 Exception이 발생하여 Connection이 끊어지는 경우, 메시지는 유실되지 않고 다른 Consumer에 의해 처리될 수 있다.

Acknowlege는 설정에 따라
* Consumer로 메시지 수신 시 자동으로 Acknowledge를 Broker에게 전달
* Consumer에서 수동으로(Programmitically) Acknowledge 처리
를 할 수 있다.

### AMQP's Entities

#### Exchange

Exchange는 메시지를 전달하는 AMQP의 개체이다. Exchange는 메시지를 수신하여 이를 0개 이상의 Queue에 전달하는 역할을 담당한다. 메시지를 Routing하는 방식은 Exchange의 종류와 Binding이라고 불리는 Rule에 의해 결정된다. 아래는 AMQP 0-9-1의 Exchange 종류이다.

| Name             | Default pre-declared names              |
|------------------|-----------------------------------------|
| Direct exchange  | (Empty string) and amq.direct           |
| Fanout exchange  | amq.fanout                              |
| Topic exchange   | amq.topic                               |
| Headers exchange | amq.match (and amq.headers in RabbitMQ) |

이와 별개로 Exchange는 몇가지 특성들을 포함하고 있다.

* Name: Exchange의 이름을 의미한다.
* Durability: Broker가 재시작해도 Exchange를 유지할지에 대한 Flag.
* Auto-delete: 모든 Queue가 Unbind 되었을 때 Exchange를 유지할지에 대한 Flag.

##### Direct Exchange

![Direct Exchange](/assets/spring/direct-exchange.jpg)

Direct Exchange는 메시지의 Routing Key를 이용해 Queue에게 메시지를 전달한다. Direct Exchange는 Unicast 방식으로 메시지를 전달할 때 유용하다.(물론 활용에 따라 Multicast 방식도 가능하다)

* Queue는 Routing Key 'K'를 이용하여 Exchange에 Binding한다.
* 메시지가 Routing Key 'R'과 함께 Exchange에 수신되었을 때, 'K'와 'R'이 일치한다면 Exchange는 메시지를 해당 Queue에 전달한다.

Direct Exchange는 여러 개의 Worker에게 작업을 분산할 때 주로 사용된다.

> 하나의 Exchange에 서로 다른 Routing Key를 가진 Queue 여러 개가 Binding 될 수 있다.
> Direct Exchange에 동일한 Routing Key를 가진 서로 다른 Queue가 Binding 된다면, 동일한 Routing Key를 가진 Queue 들에게는 Fanout 방식으로 메시지가 전달된다.

##### Fanout Exchange

![Fanout Exchange](/assets/spring/fanout-exchange.jpg)

Fanout Exchange는 Exchange에 Binding 된 모든 Queue에게 같은 메시지를 전달한다(Ignore Routing Key). 만일 N개의 Queue가 Exchange에 Binding 되어 있는 경우, 새 메시지가 Exchange에 발생한 경우 N개의 Queue에 새로운 메시지가 복사된다.

Fanout Exchange는 메시지를 Broadcasting 할 때 유용하게 사용된다.

> Topic Exchange와 Header Exchange의 경우 현재 사용할 예정이 없기에 정리하지 않음.

#### Queue

AMQP 0-9-1 Model의 Queue는 다른 Queueing System에서의 Queue 개념과 거의 유사하다. Queue는 Consumer가 소비할 메시지들을 저장하고 있는 장소이다. Queue는 몇몇 Property를 Exchange와 공유한다.

* Name: Queue의 이름이다.
* Durable: Broker 재시작 후에도 Queue를 유지할지에 대한 Flag.
* Exclusive: Queue가 1개의 Connection만 허용하고, 해당 Connection이 끊긴다면 Queue를 유지할지에 대한 Flag.
* Auto-delete: Queue를 바라보던 Consumer 들이 모두 제거된 후 Queue를 유지할지에 대한 Flag.

##### Queue Name

어플리케이션은 자신이 사용할 Queue 이름을 선택한 후 Broker에게 생성을 요청한다. Queue 이름은 255byte(UTF-8)까지 허용된다. 빈 문자열을 이용해 Queue 이름을 선언하여 생성을 요청하는 경우, Broker가 임의의 Unique한 Queue 이름을 생성한 뒤 Application에게 반환해준다.

> amq. Prefix를 가지는 Queue는 Broker 내부에서 사용되는 Queue 이름이기 때문에 생성 요청 시 403 코드를 포함한 오류가 발생하게 된다.

##### Queue Durability

Durable Queue는 Disk에 유지되며, Broker가 재시작 되어도 유지된다. Durable하지 않는 Queue는 Transient라고 불린다.

단, Queue가 Durable하다고 해서 그 안에 저장된 메시지까지 유지되는 것은 아니다. 브로커가 재시작된 이후에도 메시지를 유지하고 싶다면 메시지를 Persistent 타입으로 지정해야 한다.

#### Binding

Binding은 Exchange가 메시지를 Queue에게 전달하기 위한 규칙이다. 만일 메시지에 일치하는 Binding이 존재하지 않는 경우 메시지는 유실되거나 Publisher에게 반환된다.

#### Consumer

Consumer는 Queue에 저장된 메시지를 소비하는 어플리케이션이며, Push와 Pull 방식으로 동작할 수 있다.

* Push 방식: 메시지가 도착하면 Consumer에게 **전달**된다.
* Pull 방식: Consumer가 필요할 때 메시지를 **가져**간다.

작성중...