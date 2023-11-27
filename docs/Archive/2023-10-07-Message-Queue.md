---  
layout: default
parent: Archive
title: "Message Queue"
categories: Software
tags:
  - system
---  


Message Queue에 대해 알아보기 전에 컴퓨팅 시스템에서 Queue가 어떤 의미를 가지는지 훑고 넘어가자.
컴퓨팅 시스템에서 서로 다른 시스템 간 커뮤니케이션을 할때, 그 중간에 Queue를 두는 것이 일반적이다.

예를 들어, 하나의 CPU가 여러 쓰레드들의 작업을 받아 처리하기 위해 Job Queue나 Ready Queue를 작업 처리전 임시 저장 공간으로 사용한다. 또 물리적으로 떨어진 어플리케이션 간(클라이언트와 서버 간)에 Connection 형성을 위해  Connection Accept Queue를, 통신 데이터를 주고받기 위해 TCP I/O Buffer Queue를 사용한다.

![Queue 예시](https://github.com/kids-ground/mentos_flutter/assets/52196792/ddc7891c-090b-4749-aa5b-ffbe7b2d64a7){: .align-center style="width: 100%;"}  
컴퓨팅 시스템에서 Queue
{: .image-caption style="font-size: 14px;" }  


Queue를 사용하는 것은 컴퓨팅 시스템에서 매우 당연한 일이다. 대부분의 시스템간 상호작용이 일어나는 곳에서는 항상 Queue를 사용한다. 왜 그럴까? Queue가 존재하지 않는다고 생각해보자. 들어오는 모든 요청을 바로바로 처리 해야한다. 처리를 담당하는 자원은 하나인데 요청은 동시에 수십, 수백개가 떨어진다면 해당 자원은 꽤나 많은 수의 요청을 처리하지 못하고 버리게 된다. 이러한 상황을 방지하고 두 시스템 사이에서 요청을 잠시 보관하는 임시공간으로 Queue를 사용한다. 즉, 다시 말해 **Queue는 다량의 요청에 대한 일종의 디펜스 매커니즘인 셈**이다.

물론 Queue가 담을 수 있는 수용량 이상의 요청에 대해서는 마찬가지로 처리가 불가능하지만 큐가 수용가능한 수준내에서는 시스템이 안정적으로 요청을 처리할 수 있게 도와주는 역할을 한다.

<br />  

### Message Queue
Message Queue는 어플리케이션 간 데이터를 교환할때 사용하는 통신 방법이다. Message Queue로 메시지를 전송하는 Producer와 Message Queue에서 Message를 받아 처리하는 Consumer가 존재한다. 

![Message Queue](https://github.com/kids-ground/mentos_flutter/assets/52196792/a5e74451-613e-404b-983a-0eadfa6aef5f){: .align-center style="width: 100%;"}  
Message Queue
{: .image-caption style="font-size: 14px;" }  


Message Queue는 위에 설명한 Queue와 마찬가지로 해당 특징(다량의 요청에 대한 방어체제)을 가지면서 **큐에 저장되는 메세지를 잃어 버리지 않는다**는 특징이 있다. 대부분의 시스템에서의 큐들은 시스템이 다운되면 가지고 있던 정보를 모두 잃는 반면 Message Queue는 데이터(메세지)를 잃어버리지 않고 보관한다.

또한, Message Queue에서 메세지를 받아 처리하는 Consumer에서 데이터 처리중 문제가 발생해서 다운되어도 메세지를 보관한다는 장점이 있다. 다시 말해, **안정성이 매우 뛰어난 시스템**이다.

커머스 플랫폼에서 상품을 구매하는 상황을 가정해보자. 출금 후 상품 출고 순으로 진행되어야 하는데, 출금되고 나서 시스템이 다운된다면 진행중이던 정보가 날아가고 따라서, 상품 출고는 이뤄지지 않는 불상사가 발생한다.

만약  이 서비스가 `출금 -> Message Queue -> 상품출고`  흐름을 갖는다면 어떨까? 출금이후 상품 출고를 위해 메세지 큐로 메세지를 전송할 것이다. 상품출고 어플리케이션이 다운되어도 Message Queue가 이를 보관하고 있으므로 큰 문제가 없다. 상품출고 어플리케이션이 다시 동작하게 되면 Message Queue에서 메세지를 꺼내 처리할 수 있기 때문이다.  상품출고 어플리케이션이 메세지를 꺼내 동작하는 도중에 다운되어도 문제 없다. 처리완료 되지 않은 메시지에 대해서 Message Queue가 계속 보관하고 있기 때문이다. 

위 예시와 같이 Message Queue는 **안정성이 중요한 부분, 실패가 있어서 안되는 부분에서 주로 사용**된다. 결제 시스템, 산업 시스템, 커넥티드 카 등 안정성이 우선인 시스템에서 많이 사용된다.  

<br />  

### Message Queue 구조
앞서 Message Queue는 메세지를 잃어버리지 않고 Consumer의 처리가 완료될 때까지 보존한다고 했다. 어떻게 가능한 것일까? 구조는 간단하다. Consumer에서 메세지를 처리한 후 accept를 보내주는 방식이다. 만약 지정된 처리시간이 경과되었거나 명시적으로 rejected를 Message Queue에 응답할 경우, 해당 메세지는  Message Queue 내부에 따로 존재하는 Dead-letter Queue에 저장되어 이후 처리까지 보관한다.


![Message Queue 구조](https://github.com/kids-ground/mentos_flutter/assets/52196792/0f97119a-8964-41a3-b597-447d73972a61){: .align-center style="width: 100%;"}  
Message Queue 구조
{: .image-caption style="font-size: 14px;" }  

<br />  

### Message Queue를 사용함으로써 얻는 다양한 이점들

Message Queue는 시스템에 안정성 뿐만 아니라 몇몇 이점을 가져다 준다.
1. 비동기 처리로 이한 효율성
	- Message Queue는 메세지 지향 미들웨어(MOM. Message Oriendted Middleware)를 구현한 시스템이다. 이는 어플리케이션 사이에서 비동기로 메세지를 송수신할 수 있는 모델을 의미한다.
	  비동기 처리 매커니즘 덕분에  Producer는 Message Queue에서 메세지가 처리되기를 기다리지 않고 다음 작업을 이어나갈 수 있다. 
1. 시스템간 낮은 결합도
	- Message Queue를 사이에 두고 Producer와 Consumer의 구분이 명화해진다. Produder가 바뀌든 Consumer가 바뀌는 각 어플리케이션에 영향을 끼치지 않는다.
2. 독립된 확장성
	- Producer, Consumer모두 원하는 대로 각각 따로 규모를 확장할 수 있다. 

<br />  

### Message Broker
Message Queue의 또다른 특징은 메세지의 내용을 알 수 없으며, 하나의 메세지를 여러 Consumer가 동시에 받아 처리할 수 없다. Message Queue는 단순히 **Producer로부터 메세지를 받고 해당 메세지를 처리할 하나의 Consumer를 선택해 메세지를 전송하는 역할만을 담당**한다. 

Message Broker는 Message Queue를 관리하는 컴포넌트로 **메세지를 관리하는 방법을 제공**한다.
- Topic
	- 메시지 브로커는 Topic 기반 메시징을 지원한다. 이는 발신자가 메시지를 하나 이상의 토픽에 게시하고, 관심 있는 수신자(Message Queue)가 해당 토픽을 구독하여 메시지를 받을 수 있다. 이는 이벤트 기반 아키텍처를 구현하는 데 유용하다.
- Pub - Sub 구조
	- 발신자가 메시지를 게시하면 해당 메시지를 관심 있는 구독자(Consumer 그룹)에게 자동으로 전달한다. 이 모델은 이벤트 드리븐 아키텍처를 위한 중요한 요소이다.



참고
- [What is a Message Queue and when and why would I use it](https://www.youtube.com/watch?v=bHSV916YbHE&t=1s)