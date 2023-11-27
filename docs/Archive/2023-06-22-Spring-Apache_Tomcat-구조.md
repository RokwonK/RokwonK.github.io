---
layout: default
parent: Archive
title: "Spring - Apache Tomcat 구조"
categories: Spring
tags:
  - tomcat
---  

Spring을 이용하여 서버를 구현할 때 보통 Tomcat을 Servlet Container로 많이 이용한다. 그렇다면 Tomcat은 어떻게 클라이언트의 요청을 받아들이고 Spring 웹 어플리케이션으로 요청을 전달하게 되는걸까?  


## Apache Tomcat 구조
Tomcat Server는 하나의 JVM 위에서 실행된다.(경우에 따라선 여러개의 Instance를 실행할 수 있으며 각 Instance마다 JVM을 할당할 수도 있다.) 즉, Tomcat은 JVM 위에서 동작하는 Java 웹 어플리케이션 서버라는 의미이다.  

Tomcat은 내부적으로 여러 구성요소로 이루어져있다. 당연히 각 구성요소는 각자의 역할을 가지고 있다.  

![tomcat](https://github.com/kids-ground/mentos-backend/assets/52196792/c3536c14-059e-496e-9a30-1a61b9efddd9){: .align-center style="width: 60%;"}  
Tomcat 구조
{: .image-caption style="font-size: 14px;" }  

### Server
Java로 작성된 프로그램이다. Server를 실행시킨다는 것은 Tomcat Instance를 만든다는 것이고 즉, 톰캣 프로세스가 동작한다는 것을 의미한다.  
실행 이후에는 특정 포트로 명령을 내려 실행을 중단시킬 수 있다.  

<br />  

### Service
`Connector`를 구성하고 관리 및 제어를 하는 역할이다. 일반적으로 하나의 인스턴스에는 하나의 `Service`만을 띄운다.  
`Service`는 외부 요청을 관리하는 역할을 하는 Engine과 요청을 받아들이는 입구역할을 하는 `Connector`들을 가진다.  

<br />  

### Engine  
`Host`, `Context` 등 하위요소들을 관리 제어한다.
1. 여러개의 가상 호스트를 지원한다.(다수개의 Host를 만들고 관리)
2. `Connector`로 들어오는 요청을 보고 IP, Domain에 따라 `Host`로 분배
3. `Connector`로 들어오는 요청을 보고 라우팅 경로에 따라 `Context`로 분배  

Tomcat 9.0 버전 이후부터는 Catalina Engine을 사용한다. 보통 우리가 말하는 Servlet Container란 이 Engine을 부르는 이름이다.

<br />  

### Host
`Host`는 `Engine`의 가상호스팅에 의해 **나누어진 웹 호스팅 단위**이다. 외부 요청의 IP, Domain에 따라 다른 Host로 요청을 분배한다. 예를 들어, rokwon.com으로 들어온 요청은 Host1로 wonrok.com으로 들어온 요청은 Host2로 나눌 수 있다. 즉, 하나의 서버가 여러 IP를 가지고 IP별 독립적으로 요청을 처리할 수 있다.  

**가상호스팅의 장점과 같이 하나의 서버로 여러 IP를 운영하여 서버 자원을 절약**{: .font-highlight}할 수 있다.  

<br />  

### Context
**Context는 하나의 웹 어플리케이션**을 나타낸다. 주로 `war` 파일의 형태로 배포가 된다.  
앞서 `Host`에서 IP/Domain으로 나누어 들어온 요청을 다시 라우팅 경로로 나누어 각 `Context`로 요청이 나누어진다. 예를 들어 api/context1 은 Context1로 api/context2 는 Context2로 나누어 요청이 들어간다.  

이후부터는 우리가 흔히 아는 Spring Framework의 동작과정이 이루어진다. 들어온 요청을 처리할 적절한 Servlet(DispatcherServlet 같은)을 찾아 요청을 보내고 요청처리과정에 맞추어 데이터를 처리 후 응답한다.  

<br />  

### Connector
`Connector`는 외부와의 통신을 담당하는 객체로 특정 TCP port에서 요청을 수신한다. 요청이 들어오면 `Engine`을 통해 알맞는 `Host`, `Context`로 요청을 처리한다.  

Tomcat은 자체적으로 Web Server의 기능을 할 수 있는데 HTTP/1.1, HTTP/2.0 Connector가 Web Server의 역할을 수행할 수 있도록 지원해주기 때문이다. 또 다른 AJP Connector는 Apache HTTP WEb server의 요청을 처리하기 위한 특수한 Protocol을 처리한다.  

또한 Connector는 Connection을 처리한 방식에 따라 NIO와 BIO로 나누어진다. Tomcat 8.0부터는 NIO2가 도입되었고 Tomcat 9.0부터 BIO 방식이 없어졌다. **BIO에서 NIO방식으로의 변환은 'Thread per Connection'에서 'Thread per Request' 모델로의 변환**{: .font-highlight}을 의미한다.  

**💡 Connection과 Request**  
Connection은 소켓 연결을 의미하고 Request는 하나의 HTTP 요청을 의미한다. 일반적으로 하나의 Connection에서 1개 또는 그 이상의 Request가 존재한다.(Keep-Alive 헤더를 이용하여 한 Connection에서 다수의 Request, WebSocket을 이용하여 하나의 연결속에 지속적인 Request 등)
{: .notice}  

**💡 Thread per Connection**  
소켓 연결 이후 연결이 끝날때 까지 하나의 쓰레드를 점유하는 방식. 따라서 연결된 소켓의 갯수가 곧 쓰레드의 갯수이다. 이 방식은 수 많은 이용자가 사용하는 상황에서 커넥션이 쓰레드를 잡고 있어 쓰레드를 효율적으로 사용하지 못해 요청을 처리하지 못하는 상황이 발생한다.  
{: .notice--info}  

**💡 Thread per Request**  
하나의 요청에 하나의 쓰레드를 점유하는 방식. Selector라는 기능을 이용해 소켓(채널)을 등록하고 I/O 발생시 비동기적으로 쓰레드를 할당하는 방식. Thread per Connection 방식보다 쓰레드를 효율적으로 사용할 수 있다.  
{: .notice--info}  

NIO의 사용으로 Connection에 대해 요청을 비동기적으로 쓰레드에 할당할 수 있기 되었다. 하지만 Spring Framework가 자체가 요청을 처리하는 과정을 Blocking 방식으로 처리한다. 데이터베이스 조회, 외부 요청 등의  I/O들을 Blocking 방식으로 처리하기 때문에 응답을 받을때까지 Thread가 대기하게 된다. 이러한 문제를 해결하기 위해 Spring WebFlux와 같은 기술들이 등장하였다.  

<br />  

### 요청흐름 요약
![tomcat_flow](https://github.com/kids-ground/mentos-backend/assets/52196792/eaecf4e6-4b81-4841-8731-2a336dc2d416){: .align-center style="width: 70%;"}  
Tomcat 요청흐름
{: .image-caption style="font-size: 14px;" }  

- 커넥터가 요청을 받아들인다.(포트마다 커넥터가 다르다)
- 요청을 받은 커넥터는 IP,Domain에 따라 Host를 찾고
- 다시 라우팅 경로에 따라 Context를 찾는다.
- Context에서는 요청을 처리하고 역순으로 응답한다.  

<br />

**참고**  
- https://tomcat.apache.org/tomcat-10.0-doc/config/
- https://velog.io/@asdfdwa/%ED%86%B0%EC%BA%A3-%EC%84%9C%EB%B2%84-%EA%B5%AC%EC%A1%B0
- https://kadensungbincho.tistory.com/62
- https://girinprogram93.tistory.com/68
- https://velog.io/@jihoson94/BIO-NIO-Connector-in-Tomcat
