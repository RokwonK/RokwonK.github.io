---
layout: default
parent: Archive
title: "Spring - Apache Tomcat 스레드 관리"
categories: Spring
tags:
  - tomcat
---  


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

### Connection 처리  
Tomcat의 Connection 처리방식에는 Blocking I/O, NonBlocking I/O가 존재한다. 이에 관한 자세한 내용은 이 포스트 Connector 부분에 설명해 두었다.  
짧게 말해 Tomcat은 8.0 버전 이후부터는 NonBlocking I/O(NIO) 방식을 이용하여 Connection을 처리한다.

<br />  

### Tomcat 설정
- maxConnections
  - 최대 커넥션 갯수를 설정 
  - 다시 말해 소켓의 갯수(최대 연결될 수 있는 클라이언트 수)를 설정하는 것
- acceptCount
  - 최대 커넥션 개수를 초과하여 요청시 요청을 대기시킬 큐의 크기
  - 이 큐가 꽉차면 이후 요청은 버려짐
- maxThreads
  - 최대로 실행가능한 쓰레드 수

### 최적화하기
최대 갯수, 최소 갯수, 커넥션



**참고**
- - https://www.youtube.com/watch?v=um4rYmQIeRE&list=PLgXGHBqgT2TvpJ_p9L_yZKPifgdBOzdVH&index=66