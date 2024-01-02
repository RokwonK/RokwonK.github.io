---
title: "MySQL Datetime과 Timestamp"
categories: Database
tags:
  - mysql
---  

MySQL에는 날짜 표기법으로 `DATE`, `DATETIME`, `TIMESTAMP`가 있다.

- `DATE`  
**형식** : YYYY-MM-DD  
**범위** : 1000-01-01 ~ 9999-12-31  
**크기** : 3 byte  
- `DATETIME`
**형식** : YYYY-MM-DD hh:mm:ss  
**범위** : 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59  
**크기** : 8 byte  
- `TIMESTAMP`  
**형식** : YYYY-MM-DD hh:mm:ss  
**범위** : 1970-01-01 00:00:01 UTC ~ 2038-01-19 03:14:07 UTC  
**크기** : 4 byte  


`DATE`는 날짜만을 표기하는 것으로 확인할 수 있지만 `DATETIME`과 `TIMESTAMP`의 차이는 뚜렷하게 확인하기 어렵다. 이 둘의 차이는 무엇일까?

바로 Timezone의 적용여부이다. `TIMESTAMP` 의 경우에는 시간을 가져올때 Timezone을 적용해서 가져온다.

`DATETIME`과 `TIMESTAMP`는 기본적으로 System의 time zone을 기준으로 날짜를 보여준다. 이때 Session time zone이나 Global time zone 을 교체하면 어떻게 될까? **`DATETIME`의 경우 System time zone을 따르고 `TIMESTAMP`는 Session, Global, System 순으로 time zone을 따른다.**  

하나의 나라에서 서비스를 한다면 문제가 없겠지만 글로벌 서비스의 경우에는 나라마다 다른 timezone을 가지므로 timestamp를 사용하는게 유리하다.  


{% capture mysql %}
💡 **MySQL의 환경설정**  
MySQL은 System, Global, Session 단위로 설정을 적용할 수 있다.  
- Session : 현재 세션에만 영향을 주는 설정이다. 즉, 클라이언트가 MySQL 서버에 연결 후 종료할 때까지의 작업 내에서만 유효하다.
- Global : MySQL 서버 인스턴스 전체에 영향을 주는 설정이다. 서버 인스턴스가 실행 중인 동안에만 유지되며 서버 재시작시 이 설정도 초기화된다.
- System : MySQL 서버 시작시 읽는 전역 설정 파일에 포함된 설정이다. 서버의 기본 동작을 지정하며 만약 설정을 바꾼다면 재시작을 해야만 적용된다.  
{% endcapture %}

<div class="notice--info">{{ mysql | markdownify }}</div>  
<br/>

**참고**
- https://nesoy.github.io/articles/2020-02/mysql-datetime-timestamp