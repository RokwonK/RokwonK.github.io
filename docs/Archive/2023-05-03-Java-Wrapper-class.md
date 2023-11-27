---
layout: default
parent: Archive
title: "[Java] Wrapper Class"
categories: Java
tags:
  - wrapper
---  

기본 자료형들은 객체가 아니기 때문에 **객체를 다룰때 제공되는 유용한 메서드들을 사용하지 못한다.** 이를 보완하기 위해, 자바에서는 기본 자료형을 객체로 다룰 수 있도록 래퍼 클래스를 제공한다. 
- int -> Integer
- char -> Character
- boolean -> Boolean  

<br />  

### 언제 사용할까?
내장 메서드를 이용할때 뿐만 아니라 다음과 같은 상황에서 유용하게 사용할 수 있다.  

1. **null 값 저장**
  - 기본 자료형은 null값을 허용하지 않음. 따라서, null값을 사용해야하는 경우 래퍼 클래스를 사용
2. **Collection(컬랙션)에 기본 자료형 값을 저장할때(자동변환)**
  - 컬렉션은 객체만 저장할 수 있기 때문에 기본자료형을 저장할려면 래퍼 클래스를 사용
3. **제네릭 타입을 파라미터로 받을때**
  - 기본자료형의 래퍼클래스들도 `Object`를 상속받는다.

<br />

### 자동변환
래퍼클래스는 기본 자료형에 도움을 주기위한 클래스이기 때문에 상호간 쉽게 호환될 수 있도록 자동변환이 이루어질 수 있다. 다음은 자동변환이 이루어지는 상황들이다.

1. 메서드 인자로 전달할 때
2. 산술 연산
3. 비교 연산


```java
public static void main(String[] args) {
    Integer integer = 10;

    // 메서드로 전달
    changeValue(integer);

    // 산술연산
    int i = integer + 30;

    // 비교 연산
    if (i == 10) {

    }
}

public static void changeValue(int v) {
    // ...
}
```


**참고**
- [Java Wrapper 클래스](https://junhyunny.github.io/java/java-wrapper-class/)