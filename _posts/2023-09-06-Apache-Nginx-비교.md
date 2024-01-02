---  
title: "Nginx. Apache와 비교하며.."
categories: Software
tags:
  - server
---  


현재(2023년 9월 기준) 가장 인기있는 웹서버가 무엇이냐고 물어보면 단언컨대 Nginx가 빠질 수 없다. Nginx는 2004년 출시 이후 꾸준히 웹서버 시장에서 점유율을 높이기 시작했다. 2008년 모바일 시장이 본격화됨에 따라 급부상하여 현재에 이르러 시장 점유율 34% 기록하고 있으며 유튜브, 메타 등 알만한 글로벌 IT 기업에서도 Nginx 사용하고 있다. Nginx에 어떤 매력이 있기에 수 많은 기업들의 선택을 받은 것일까?  

### 웹서버의 역할
앞으로 살펴볼 Nginx는 웹 서버 서비스로 잘 알려져 있다. 본격젹으로 Nginx에 대해 알아보기 전에 현대의 웹 서버의 역할에 대해서 간단하게 살펴보자. 웹 서버의 사전적 정의는 *'HTTP 또는 HTTPS를 통해 클라이언트에서 요청하는 HTML 문서나 오브젝트(이미지 파일 등)를 전송해주는 서비스 프로그램'*{: .gray} 이다.([나무위키 참고](https://ko.wikipedia.org/wiki/%EC%9B%B9_%EC%84%9C%EB%B2%84))

하지만 모바일 시대가 시작되고 인터넷 트래픽이 급증하면서 웹 서버에게 요구하는 기능들이 많아졌고 그에 맞춰 현대의 웹 서버 서비스들에서도 다양한 기능들을 제공해주고 있다. 이러한 기능들은 **보안이나 안정성, 성능 향상을 위해 존재**한다.  

보안에 있어서 대표적으로 **리버스 프록시** 역할을 담당 할 수 있다. 이를 통해 웹 어플리케이션의 실제 IP를 숨기고 웹 어플리케이션을 보호하고 외부의 공격으로부터 방어할 수 있다. 또한 보안의 기능으로써 IP별로 **처리율을 제한**하거나 간단한 **인증처리를 수행하는 역할**을 할 수도 있다.  

![리버스 프록시, 처리율 제한, 인증처리](https://github.com/kids-ground/shout-backend/assets/52196792/1c3532cb-fd4a-4b77-a83d-996f3965d8f7){: .align-center style="width: 70%;"}  
리버스 프록시로서의 역할
{: .image-caption style="font-size: 14px;" }  

성능 향상의 측면에서는 **캐싱기능이나 컨텐츠 압축 전송의 역할**을 할 수 있다. 웹 어플리케이션에서의 응답을 웹 서버에서 캐싱하고 같은 요청이 들어오면 웹 서버에서는 웹 어플리케이션에 접근하지 않고 캐싱된 데이터를 바로 응답할 수 있다. 또한 압축 전송을 통해 응답되는 데이터의 크기를 줄여 응답 속도를 높일 수 있다.  

![캐싱 기능, 압축 전송](https://github.com/kids-ground/shout-backend/assets/52196792/5102b678-150e-44e2-aae6-2782152751bc){: .align-center style="width: 70%;"}  
캐싱, 압축 전송 기능
{: .image-caption style="font-size: 14px;" }  

안정성을 위한 기능으로써는 가장 유명한 **로드 밸런서 역할**을 예로 들 수 있다. 로드 밸런서는 요청 트래픽을 뒷단의 여러 웹 어플리케이션으로 분산시켜줄 수 있는 기능이다. 하나의 서버가 동시에 처리할 수 있는 요청 수에는 한계가 있기 때문에 트래픽이 과도하게 몰리게 되면 서버가 중단되는 문제가 발생 할 수 있다. 때문에 스케일 아웃을 통해 서버의 수를 늘리고 각 서버로 균등하게 트래픽을 분산시켜 줘야한다. 웹 서버에는 균등 분배 알고리즘이 몇 가지 존재하며 해당 알고리즘을 통해 트래픽을 분산시켜준다.

이렇듯 현대의 웹서버는 클라이언트의 요청에 대해 요청 메시지를 이용한 처리 이외의 여러가지 부가기능들을 담당하여 보안, 안정성, 성능향상 등 시스템의 전반적인 퍼포먼스를 올려주는 필수 불가결한 서비스가 되었다.  

<br />  

### Nginx의 등장 - Apache C10K 문제
Nginx가 출시되기 이전에는 Apache가 웹 서버 생태계를 이끌어 가고 있었다. 그러던 와중, IT 인프라가 발전하고 인터넷에서의 트래픽이 나날이 늘어나면서 Apache에서 큰 문제가 발견되었다. 바로 **C10K(Concurrent Connections 10,000) 문제**이다. C10K란 **동시에 10,000개의 연결을 처리할 수 있는 능력**을 가리킨다. Apache에서는 어째서 이러한 문제가 생겼던 것일까? 답은 Apache의 고질적인 동작구조에 있다.  

Apache는 멀티 프로세스 방식으로 동작한다. Connection이 들어오면 하나의 Process를 만들고 해당 Connection을 담당한다. 즉, **Connection과 Process가 1:1 관계**를 맺는다. 물론, Process 생성 비용은 매우 비싸기 때문에 일정량의 Process를 미리 만들어 놓고 Connection이 들어올때 만들어 놓은 Process를 가져다 쓰는 Prefork 방식을 사용한다.  

![멀티 프로세스 Prefork 방식](https://github.com/kids-ground/shout-backend/assets/52196792/cc7a3d85-feaa-4891-a1f2-9ce510f74953){: .align-center style="width: 50%;"}  
멀티 프로세스 Prefork 방식  
{: .image-caption style="font-size: 14px;" }  

Prefork 방식으로 Process 생성비용을 아낄 수 있었지만 멀티 프로세스 방식 자체가 몇 가지 고질적인 문제를 초래한다. 프로세스간 **Context-Switching은 비용이 많이 들며 CPU에 큰 부하**를 준다. 또한 Process 개수가 늘어남에 따라 **많은 메모리를 사용하게 되고 메모리 부족 현상**이 일어날 수 있다. 이러한 문제들은 동시 커넥션 수 10,000개가 넘어갈때 도드라지고 커넥션을 잃어버리는 문제가 빈번하게 일어났다.(C10K 문제 발생)  

Nginx는 이러한 C10K 문제를 보완하고자 등장하게 되었다. Apache의 구조적인 한계를 극복하고 수 백만의 동시 커넥션도 버틸 수 있는 구조로 설계되었다.  

<br />  

### Nginx Connection 처리구조
NginX는 **비동기-논블로킹 이벤트 기반구조**{: .font-highlight}로 만들어졌다. 쉽게 말해 요청(이벤트) 처리 시 블로킹하지 않고 처리함으로써 컴퓨팅 자원을 효율적으로 활용할 수 있도록 만든 구조이다. 정의만 보고 감을 잡기가 쉽지 않다. 차근차근 살펴보도록 하자.

우선, Nginx가 처음 실행될 때 무슨 일이 일어나는지 살펴보자. Nginx는 처음 실행 시 하나의 `Master Process`가 동작한다. `Master Process`는 Nginx의 관제탑 역할을 담당한다. Master Process의 역할을 나열해보자면 다음과 같다.  

> - nginx 설정 파일 읽기
- 포트를 열고 socket을 생성, 바인딩, 닫기
- **위 과정 진행 후 `Worker Process` 생성**  
- 서비스 중단없이 재구성 실행

`Master Process`는 실행되면 설정 파일(`nginx.conf`)을 읽고 정보에 맞춰 Listening Port를 열고 Listen Socket을 만들어 Port에 바인딩 한다. 이 후 설정 파일에 정의한 수(보통 CPU 코어 수) 만큼 `Worker Process`을 생성한다. **`Worker Process`는 실제로 클라이언트의 연결, 요청을 처리하는 역할을 맡는 작업 프로세스**이다. `Worker Process`가 클라이언트의 요청을 받을 수 있도록 `Master Process`는 생성한 Socket들을 각 `Worker Process`에 배정한다.  

![Master Process의 동작](https://github.com/kids-ground/mentos-backend/assets/52196792/a60bc99d-a561-46c6-9cf2-78c0a053a5a0){: .align-center style="width: 50%;"}  
Master Process와 Worker Process
{: .image-caption style="font-size: 14px;" }  

물론 `Master Process`와 `Worker Process` 외에도 `Cache-Loader Process`와 `Cache-Manager Process`도 존재하지만 요청 처리 구조를 설명하는데 함께 이야기하면 복잡해지므로 잠시 생락한다. 이렇게 하면 클라이언트 요청을 받기 위한 준비 작업이 완료된다.  

이제 연결요청을 처리하는 구조를 살펴보자. 클라이언트의 연결요청이 Port를 통해 들어오면 OS 커널은 연결된 Socket 중 적당한 하나를 골라 들어온 요청을 보낸다.(OS 커널은 보통 라운드 로빈 알고리즘을 통해 각 소켓에 연결을 분배한다) `Worker Process`는 **자신에게 배정된 Socket으로 이벤트가 들어오면 해당 이벤트를 처리**한다. 여기서 이벤트란, Socket으로부터 들어오는 Connection 할당, Request 등 모든 요청을 의미한다. **`Worker Process`는 들어온 이벤트들을 큐 형태로 쌓아 하나씩 작업을 처리**한다.  

`Worker Process`는 **싱글 스레드 내에서 들어온 이벤트들을 큐에 쌓고 이벤트 루프가 큐의 값을 꺼내 비동기 처리가 필요한 것들을 블로킹하지 않고 비동기 I/O**{: .font-highlight}(OS 커널 수준의 매커니즘을 이용. 즉, 커널에 맡김) 처리 한다. 덕분에 요청을 처리하는데 멈춤없이 다음 작업을 처리할 수 있다.  

![이벤트 큐와 이벤트 루프](https://github.com/kids-ground/mentos-backend/assets/52196792/3046acbd-1753-41bd-b58d-7c13dd27547c){: .align-center style="width: 40%;"}  
Worker Process의 이벤트큐와 이벤트 루프
{: .image-caption style="font-size: 14px;" }  

Nginx의 장점은 **싱글 스레드로 동작하며 요청을 논블로킹 처리한다는 것**{: .font-highlight}에 있다. Apache 같은 경우 커넥션당 Process를 생성한다. 때문에 하나의 CPU 코어가 여러개의 Process를 담당하고 Process간 Context Switching 비용이 많이 발생한다. 하지만 **Nginx는 보통 코어 당 하나의 Worker Process가 붙고 Worker Process는 싱글 스레드로 동작하여 Context Switching 비용이 거의 들지 않는다.**  

또한 비동기 요청을 논블로킹으로 처리함으로써 들어오는 요청을 막힘없이 처리할 수 있다.

![싱글 스레드 처리의 장점](https://github.com/kids-ground/mentos-backend/assets/52196792/3e459626-2fd0-4526-bedc-cd00aa651dab){: .align-center style="width: 70%;"}  
멀티 프로세스 vs Nginx의 요청처리
{: .image-caption style="font-size: 14px;" }  

<br />  

### Apache의 노력 Apache MPM
Apache 또한 동시 처리 성능 향상을 위해 여러가지 노력을 기울이고 있다. Apache MPM(Multi-Processing Module)은 그러한 노력의 일환으로 등장하였고 현재 기존의 Prefork 방식과 함께, Worker, Event 방식을 지원하고 있다. Apaceh와 관련된 자세한 설명은 [링크](https://camelsource.tistory.com/71)로 대신한다.


<br />  

**참고**  
- [Inside NGINX: How We Designed for Performance & Scale](https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/)
- [The Architecture of Open Source Applications (Volume 2) nginx](https://aosabook.org/en/v2/nginx.html)
- [피케이의 Nginx](https://www.youtube.com/watch?v=6FAwAXXj5N0&t=6s)