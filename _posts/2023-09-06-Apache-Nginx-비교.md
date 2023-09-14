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

Nginx는 이러한 C10K 문제를 보완하고자 등장하게 되었다. Apache의 구조적인 한계를 극복하고 수 많은 동시 연결도 버틸 수 있는 구조로 만들어졌다.  

<br />  

### Nginx 구조
NginX는 **비동기 이벤트 기반구조**{: .font-highlight}로 만들어졌다. 쉽게 말해 요청(이벤트)을 비동기로 처리함으로써 컴퓨팅 자원을 효율적으로 활용할 수 있도록 만든 구조이다. 정의 자체가 어려운 단어의 연속이니 차근차근 구조를 알아가며 살펴보자.  

Nginx 내부 구조에는 Master Process라고 불리는 관제탑이 존재한다. Master Process는 Nginx의 설정에 따라 Worker Process를 만드는 역할을 담당한다. Worker Process는 클라이언트의 Connection과 Request를 받아 처리하는 작업 프로세스이다. 

![master, worker Process 보여주기](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/b81ab8e9-d391-4ed0-bdf1-c7ea9191897f){: .align-center style="width: 70%;"}  
Master Process와 Worker Process의 관계
{: .image-caption style="font-size: 14px;" }  

일전에 Apache에서는 Connection당 Process가 붙어 작업을 처리하였지만 Nginx에서는 Process의 수가 고정적(보통 CPU 코어의 갯수와 같다)이다. 뿐만 아니라 하나의 Worker Process가 다수의 Connection을 담당한다.

![OS 커널 Queue를 이용하는 모습, 처리시간이 긴 요청을 처리하는 쓰레드를 던지는 모습](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/7d514b28-ba0c-4088-93ed-1e03a46a2eb1){: .align-center style="width: 70%;"}  
Nginx가 이벤트를 처리하는 방식
{: .image-caption style="font-size: 14px;" }  

이러한 방식은 Process로 인한 메모리의 낭비(Process는 생성될 때마다 그 정보가 메모리에 쌓인다.)가 적을 뿐만 아니라 CPU의 유휴시간을 극적으로 줄일 수 있다. CPU의 관점에서 Apache의 멀티프로세스 방식과 Nginx의 비동기 이벤트 구조의 차이를 보면 CPU의 유휴시간을 최소한으로 줄여 더 많은 작업을 처리하는 것을 알 수 있다. 

![CPU 관점에서 보는 비동기 이벤트 처리의 장점]()


<br /> 

### Apache의 노력 Apache MPM
2008년 모바일 시대. 동시 커넥션을 많이 생성하는 계기. 뿐만 아니라 브라우저에서도 성능향상을 위해 병렬적으로 커넥션 생성.

Apache 또한 MPM(Muli processing module)이라는 모듈을 추가해서 성능 개선. 운영을 선택할 수 있음. prefork 또는 워커 프로세스 만들기. 그래도 NginX가 크게 앞선다.

Apache가 Nginx에 비해 가지는 장점
- 다양한 OS에 안정적(NginX는 윈도우에 안정적이지 못함)
- 모듈로 기능을 계속 추가할 수 있음.(확장성이 좋음)

<br />  

**참고**  
- https://www.youtube.com/watch?v=6FAwAXXj5N0&t=6s
- http://www.opennaru.com/jboss/apache-prefork-vs-worker/
- https://applefarm.tistory.com/137
- [Nginx Blog](https://www.nginx.com/blog/inside-nginx-how-we-designed-for-performance-scale/)
- https://aosabook.org/en/v2/nginx.html