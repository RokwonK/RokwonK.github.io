---
title: "[Java] String(문자열) 조작하기"
categories: Java
tags:
  - 문자열
---  

Java에서는 문자열을 표현하기 위해 여러가지 클래스들을 제공하며, 문자열을 조작하기 위해 많은 메서드들을 지원해준다. 자세히 살펴보자.  

<br />  

### String, StringBuffer, StringBuilder
Java 에서는 문자열을 저장, 관리하는 클래스들로 `String`, `StringBuilder`, `StringBuffer`가 있다. 각 클래스들은 저마다 특징을 가지고 있다.  

`String`은 **저장한 데이터가 불변**하는 특징을 지닌다. 여기서 말하는 불변이란, 메모리에 저장된 값을 수정할 수 없다는 뜻이다. 그렇다면, 해당 변수에 재할당하지 못한다는 것인가? 아니다. 재할당시 새로운 문자열 객체가 만들어지고 해당 변수는 새로운 문자열 객체를 가리키게 된다. 이때, 기존 문자열은 GC의 수거대상이 된다.  

`String`은 특이하게 리터럴 생성과 new를 이용한 생성 2가지로 나뉘어진다. 두 생성법 모두 JVM의 Heap 영역(Java7 이후)에 저장된다. 하지만, 리터럴 생성법이 메모리 측면에서 효율이 좋다. new를 이용항 생성은 일반적인 Heap 영역에 생성되지만, **리터럴 방식은 Heap 영역 중 String Constant Pool 이라고 불리는 공간에 생성**된다. 이 영역은 특이하게 동일한 문자열일 경우에 같은 주소값을 가리키게 된다. 즉, 만들어진 문자열의 주소를 재사용하는 것이다.  
```java
public static void main(String[] args) {
    String newStr = new String("new");
    String newStr2 = new String("new");

    String literalStr = "리터럴";
    String literalStr2 = "리터럴"; // 같은 메모리 주소를 가짐

    System.out.println(newStr == newStr2); // false
    System.out.println(literalStr == literalStr2);  // true, 메모리 주소가 같으므로
}
```  

`StringBuilder`는 `String`과 다르게 가변적이다. append, delete 등의 메서드들을 이용하여 문자열을 변경하는 것이 가능하다. 따라서 수정연산이 많은 경우 사용하기 좋다. 하지만, **`StringBuilder`는 Thread-safe을 보장하지 않는다.** 즉, 수정이 끝나기 전에 접근하여 예상치 못한 값을 참조할 수도 있다는 뜻이다. 이러한 동시성 문제를 제어할 수 있는 클래스가 `StringBuffer` 클래스이다. `StringBuffer`는 문자열에 대해 여러 연산이 가능하며, 동시성 제어도 가능하다. 또한, `StringBuilder` 클래스의 메서드들과 호환된다.  

```java
public static void main(String[] args) {
    StringBuilder strBuilder = new StringBuilder("String");
    strBuilder.append("Builder");
    strBuilder.delete(0, 6);

    System.out.println(strBuilder);
}
```

정리하자면, `String` 클래스의 경우 문자열 연산이 적고, 자주 조회하는 경우에 사용하면 좋다. 특히, 불변성이라는 특징덕분에 멀티 쓰레드 환경에서 문제가 발생하지 않는다.  
`StringBuilder`는 가변적이기 때문에 문자열 연산이 빈번한 경우에 사용하기 좋다. 하지만 동기화를 지원하지 않기 때문에 멀티 쓰레드 환경에서 사용을 지양해야 한다.  
반면 `StringBuffer`의 경우 가변적이면서 동기화를 지원한다. 하지만 기초적인 연산에서 `StringBuilder`보다 느리기 때문에 상황에 맞춰 사용할 필요가 있다.

<br />  

### 문자열 다루기  
`StringBuilder`의 경우, 가변적이기 때문에 문자열 내부의 값을 수정하는데 O(1)의 시간이 걸린다.(replace의 경우 더 걸릴 수 있음) 하지만 `String`의 경우 새로운 문자열을 만들기 때문에 O(N)의 시간이 걸리기 때문에 조심하는게 좋다.  

문자열을 다루다보면 문자열을 나누어야하는 상황이 발생하는데 이때 `String` 클래스의 `split` 메서드를 사용할 수도 있고 `StringTokenizer` 클래스를 이용할 수도 있다. `split` 메서드의 경우, String 배열을 리턴하며 나눌 구분자를 문자열로 받는다. `StringTokenizer` 클래스는 구분자를 문자열로 받지만 문자열에 들어가있는 모든 문자를 구분자로 이해한다. 또한, `nextToken` 메서드를 이용하여 차례대로 순회가 가능하다. 아래의 예시로 두 방법의 차이를 확인해보자.  

```java
public static void main(String[] args) {
    String str = "String";
    StringBuilder strBuidler = new StringBuilder(str);

    // 순회
    for (int i = 0; i < strBuidler.length(); i++ ) {
        // 접근
        char c = strBuidler.charAt(i);
        System.out.println(c);
    }

    // 수정
    strBuidler.setCharAt(0,'s');
    strBuidler.replace(0, 3, "STR");

    // 부분문자열
    System.out.println(strBuidler.substring(0,3)); // STR

    // 문자열 쪼개기 & 합치기
    String v = "aba-bcb-cac";
    String[] s = v.split("-c");
    Arrays.stream(s).forEach(System.out::println); // ["aba-bcb", "ac"]
    System.out.println(String.join("-c", s)); // "aba-bcb-cac"

    // StringTokenizer 사용
    StringTokenizer strToken = new StringTokenizer(v, "-c");
    while (strToken.hasMoreTokens()) {
        System.out.println(strToken.nextToken()); // aba, b, b, a
    }
}
```  

**참고**  
- [JAVA String, StringBuffer, StringBuilder](https://jeong-pro.tistory.com/85?category=793347)
- [Java에서 문자열의 특정 인덱스에 있는 문자 바꾸기](https://www.techiedelight.com/ko/replace-character-specific-index-java-string/)
- [[JAVA] String 문자열 자르기/치환하기 - split(), substring(), replace()](https://velog.io/@yanghl98/java-%EB%AC%B8%EC%9E%90%EC%97%B4-%EC%A1%B0%EC%9E%91)
- [[Java] 문자열(String) 조작 함수 이해하기 : 조작 및 비교 함수](https://adjh54.tistory.com/103)
- [Java String Pool 이란?](https://velog.io/@jeb1225/JAVA-String-Pool)
- [[Java] String, StringBuffer, StringBuilder 차이점](https://barbera.tistory.com/45)