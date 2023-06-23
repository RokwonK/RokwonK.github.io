---
title: "Spring - Apache Tomcat 구조와 스레드 풀 관리"
categories: Spring
tags:
  - tomcat
---  

Spring을 이용하여 서버를 구현할 때 보통 Tomcat을 사용한다. 그리고 이러한 Tomcat을 Servlet Container라고 부른다. Tomcat과 Servlet Container은 완전히 동일한 의미인가? 이 의문을 해결하기 위해 Apache Tomcat의 구조에 대해 살펴보자.  


## Apache Tomcat 구조
Tomcat Server는 하나의 JVM 위에서 실행된다.(경우에 따라선 여러개의 Instance를 실행할 수 있으며 각 Instance마다 JVM을 할당할 수도 있다.) 즉, Tomcat은 JVM 위에서 동작하는 Java 웹 어플리케이션 서버라는 의미이다.  

- Server
  - 일반적으로 하나의 Service를 갖는 하나의 Instance가 실행됨
  - 인스턴스라 함은 그냥 독립적으로 돌아가는 톰캣 프로세스라고 생각하면 됨
  - 즉, 톰캣을 실행한 것
- Service
  - Connector를 구성하고 관리 제어하는 역할
  - 하나의 Service에는 다수의 Engine이 존재할 수 있음
- Engine(Catalina, 카탈리나로 서블릿 컨테이너가 동작하는 곳)
  - 하나의 Engine에는 여러개의 Host를 가질 수 있음
- Host
  - 하나의 Host에는 여러개의 컨텍스트를 가질 수 있음
- Context
  - 
- Connector(Coyote, 코요태로 웹서버)
  - 외부와의 통신을 담당
  - 요청을 수신하고 Tomcat 내부로 전달
- Valve


### Connector
BIO Connector(Blocking IO)  
NIO Connector(Non-Blocking IO)  


### 요청 플로우
- 커넥터가 요청을 받아들인다.(포트마다 커넥터가 다르다)
- 요청을 받은 커넥터는 자신이 속한 서비스의 엔진으로 요청을 전달함
- 엔진은 다시 알맞은 호스트를 선택해 전달
- 호스트는 경로에 따라 적절한 컨텍스트에 전달
- 요청을 처리하고 역순으로 응답

<br /><br />  

## Tomcat의 쓰래드 처리방식
Tomcat은 클라이언트의 요청에 대한 처리를 멀티스레드를 이용하여 처리한다. 그렇다면 요청이 들어올때마다 스레드를 생성하고 응답을 한 이후에는 스레드를 제거하는 방식일까?  

### 요청 시 스레드 생성의 문제
- 처리시간 증가 - 유저 스레드 : OS 스레드 = 1 : 1
- CPU, 메모리 오버헤드 - 스레드 무한 생성

### 스레드 풀
- 작업 큐 - 요청이 들어오면 쌓이는 것
- 작업 스레드 - 작업 큐에서 요청을 꺼내 처리하는 스레드들
- 사용한 스레드를 재사용

### 스레드 컨트롤
- maximuPoolSize : 최대 스레드 갯수
- corePoolSize : 최소로 존재할 스레드 갯수
설정한 시간(keepAliveTime)동안 사용되지 않으면 요청을 처리하기 위해 생성된 스레드는 삭제된다. 이때 최소한 corePoolSize 만큼의 스레드는 남겨놓는다.

### Blocking I/O, NonBlocking I/O
Tomcat 8.0부터는 NonBlocking I/O가 추가되었다.  
Blocking I/O는 Connection 당 하나의 스레드가 담당하지만 NonBlocking I/O에서는 하나의 스레드가 여러개의 Connection을 담당할 수 있다.  

### 최적화하기
최대 갯수, 최소 갯수, 커넥션

<br /><br />  

**참고**  
- https://velog.io/@hyunjae-lee/tomcat1
- https://velog.io/@jihoson94/BIO-NIO-Connector-in-Tomcat
- https://www.youtube.com/watch?v=um4rYmQIeRE&list=PLgXGHBqgT2TvpJ_p9L_yZKPifgdBOzdVH&index=66