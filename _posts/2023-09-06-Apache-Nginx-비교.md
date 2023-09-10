---  
title: "Nginx. Apache와 비교하며.."
categories: Software
tags:
  - server
---  


현재(2023년 9월 기준) 가장 인기있는 웹서버가 무엇이냐고 물어보면 단언컨대 Nginx가 빠질 수 없다. Nginx는 2004년 출시 이후 꾸준히 웹서버 시장에서 점유율을 높이기 시작했다. 2008년 모바일 시장이 본격화됨에 따라 급부상하여 현재에 이르러 시장 점유율 34% 기록하고 있으며 유튜브, 메타 등 알만한 글로벌 IT 기업에서도 Nginx 사용하고 있다. Nginx에 어떤 매력이 있기에 수 많은 기업들의 선택을 받은 것일까?  

### 웹서버의 역할
본격적으로 Nginx를 알아가기 전에 웹 서버의 역할에 대해서 간단하게 살펴보자. 잘 알려진 웹서버의 역할에는 정적 컨텐츠 처리가 있다. 보통 정적인 컨텐츠는 웹 서버가 처리하고 동적인 요청은 WAS가 처리하는 구조로 익히 알고 있을 것이다. 오늘 날의 웹서버는 정적 컨텐츠 처리 뿐만 아니라 아래와 같은 다양한 기능들을 제공한다.  

> - 리버스 프록시
- Security
- 애플리케이션 로드밸랜서
- 캐싱
- 압축(전송속도 향상)

<br />  

### Apache와 C10K 문제
Nginx가 출시되기 이전에는 Apache가 웹 서버 생태계를 이끌어 가고 있었다. 그러던 와중, IT 인프라가 발전하고 네트워크 요청과 연결이 나날이 늘어나면서 Apache에서 큰 문제가 발견되었다. 바로 **C10K(Concurrent Connections 10,000) 문제**이다. C10K란 **동시에 10,000개의 연결을 처리할 수 있는 능력**{: .font-highlight}을 가리킨다. Apache에서는 어째서 이러한 문제가 생겼던 것일까? 답은 Apache의 고질적인 동작구조에 있다.  

Apache는 멀티 프로세스 방식으로 동작한다. Connection이 들어오면 하나의 Process를 만들고 해당 Connection을 담당한다. 즉, **Connection과 Process가 1:1 관계**를 맺는다. 물론, Process 생성 비용은 매우 비싸기 때문에 일정량의 Process를 미리 만들어 놓고 Connection이 들어올때 만들어 놓은 Process를 가져다 쓰는 Prefork 방식을 사용한다.  

![멀티 프로세스 방식 처리](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/f2e74657-60b8-46c6-bf4f-1cd88fd24425){: .align-center style="width: 70%;"}  
멀티 프로세스(prefork 방식)
{: .image-caption style="font-size: 14px;" }   

Prefork 방식으로 Process 생성비용을 아낄 수 있었지만 멀티 프로세스 방식 자체가 몇 가지 고질적인 문제를 초래한다. 프로세스간 **Context-Switching은 비용이 많이 들며 CPU에 큰 부하**를 준다. 또한 Process 개수가 늘어남에 따라 **많은 메모리를 사용하게 되고 메모리 부족 현상**이 일어날 수 있다. 이러한 문제들은 동시 커넥션 수 10,000개가 넘어갈때 도드라지고 커넥션을 잃어버리는 문제가 빈번하게 일어났다.(C10K 문제 발생)  

![연결 수 10,000 개를 넘었을때 문제 사진](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/8928cd7c-f06a-4b19-beff-6a0cf7846863){: .align-center style="width: 70%;"}  
c10k
{: .image-caption style="font-size: 14px;" }   

Nginx는 이러한 C10K 문제를 보완하고자 등장하게 되었다. Apache의 구조적인 한계를 극복하고 수 많은 동시 연결도 버틸 수 있는 구조로 만들어졌다.  

<br />  

### Nginx 구조
NginX는 **비동기 이벤트 기반구조**{: .font-highlight}로 만들어졌다. 쉽게 말해 요청(이벤트)를 비동기로 처리함으로써 컴퓨팅 자원을 효율적으로 활용할 수 있도록 만든 구조이다. 정의 자체가 어려운 단어의 연속이니 차근차근 구조를 알아가며 살펴보자.  

Nginx 내부 구조에는 Master Process라고 불리는 관제탑이 존재한다. Master Process는 Nginx의 설정에 따라 Worker Process를 만드는 역할을 담당한다. Worker Process는 클라이언트의 Connection과 Request를 받아 처리하는 작업 프로세스이다.

![master, worker Process 보여주기](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/b81ab8e9-d391-4ed0-bdf1-c7ea9191897f){: .align-center style="width: 70%;"}  
Nginx 구조
{: .image-caption style="font-size: 14px;" }  

일전에 Apache에서는 Connection당 Process가 붙어 작업을 처리하였지만 Nginx에서는 Process의 수가 고정적(보통 CPU 코어의 갯수와 같다)이다. 뿐만 아니라 하나의 Worker Process가 다수의 Connection을 담당한다.

Apache에서는 하나의 Connection 내 처리시간이 긴 요청이 있더라도 하나의 Process가 하나의 Connection을 끝까지 담당하기 때문에 유휴시간이 존재하게 된다. 하지만 Nginx에서는 네트워크, I/O 작업이 있다면 전문으로 처리하는 Thread Pool로 넘기고 다음 요청을 처리하거나 새로운 Connection을 할당받는다.

![OS 커널 Queue를 이용하는 모습, 처리시간이 긴 요청을 처리하는 쓰레드를 던지는 모습](https://github.com/AUSG/2023-No-Remember-Yes-Record/assets/52196792/7d514b28-ba0c-4088-93ed-1e03a46a2eb1){: .align-center style="width: 70%;"}  
Nginx가 이벤트를 처리하는 방식
{: .image-caption style="font-size: 14px;" }

<br /> 

### NginX의 다양한 기능들
- SSL 터미네이션(클라이언트와는 SSL 통신, 서버와는 http 통신) - SSL 복호화 과정을 NginX가 담당
- 캐싱(전달하는 데이터를 캐싱할 수 있다)
- HSTS, CORS 처리, TCP/UDP 커넥션 부하 분산, HTTP2 등 지원  


**참고**  
- https://www.youtube.com/watch?v=6FAwAXXj5N0&t=6s
- http://www.opennaru.com/jboss/apache-prefork-vs-worker/
- https://applefarm.tistory.com/137