---
title: "[Java] class와 main함수(ft. 접근제어자)"
categories: Language
tags:
  - java
---


자바로 개발을 시작한다면 class를 만들고 main함수를 작성하는 필수적인 코스를 거친다. 이 두가지는 무슨 의미가 있는 알아보자.  

### class
class는 객체를 생성하기 위한 설계도이며 내부에 상태와 행위가 존재한다. 상태로써 인스턴스 변수, 클래스 변수를 가지고 행위로써 인스턴스 메서드, 클래스 메서드를 가진다. 이 외에도 인스턴스를 생성해주는 생성자를 가지고 있다.  

**💡 객체지향 프로그래밍**  
**상태와 행동을 가진 객체들**이 각자 맡은 바 역할과 책임을 가지고 **유기적인 협력을 통해 이루어지는 프로그래밍**이다. 현실세계의 자동차를 예로 들어 볼 수 있다. 자동차를 이루는 각 부품들이 각자의 역할을 하고 주위의 부품과 협력하여 자동차로써 동작하는 것은 각 객체들이 협력하여 프로그램을 동작시키는 것과 같은 이치로 바라볼 수 있다.
{: .notice--info}  

예시를 보면서 class의 사용법을 알아보자. `class` 키워드를 이용하여 class를 선언하고 내부에 `age`, `weight` 필드(상태)와 `walk()`라는 행위(메서드)를 정의하였다. class 이름과 동일한 `Person 메서드`는 생성자로써 Person을 이용하여 객체를 생성할때 사용한다. 
```java
// ~/src/Person.java
public class Person {
    private int age;
    private int weight;

    // 생성자
    Person(int age, int weight) {
        this.age = age;
        this.weight = weight;
    }

    // 메서드
    public void walk() {
        weight -= 1;
    }
}
```

```java
// ~/src/Main.java
public class Main {
    public static void main(String[] args) {
        Person person = new Person(20, 80);
        person.walk();
    }
}  
```  

<br />  

### main 함수
Main 클래스를 보면 특이하게 생긴 함수가 보인다. `public static void main(String[] args)` 이 함수는 가장 처음 실행되는 함수로써 프로그램의 엔트리 포인트(진입점) 역할을 한다. 이는 JVM이 정해놓은 규칙으로 수정할 수 없다. main 함수의 파라미터 `args`는 커맨드라인을 통해 호출시 전달되는 파라미터가 배열로 들어간다.  

**🧐 main 함수 앞에 붙은 static을 사용하지 않는다면?**  
`static` 키워드가 붙은 변수나 메서드는 프로그램이 처음 실행될때 JVM의 Data Area 중 Method Area에 저장되고 프로그램이 종료될때까지 유지된다. JVM은 여기서 실행할 main 함수를 찾게 되는데 만약 `static` 키워드를 붙이지 않았다면 메모리에 올라가 있지 않을 것이다. 결국 JVM이 진입점을 찾지 못하고 프로그램이 실행되지 않는다. 
{: .notice}  

**🧐 main 함수 앞에 붙은 public을 사용하지 않는다면?**  
public 키워드는 접근제어자의 한 종류로 모든 패키지에서 접근 가능하다는 의미를 가진다. 만약 public으로 전체 공개 사용하지 않는다면 JVM이 접근하는데 제한이 걸려 결과적으로 프로그램이 실행되지 않는다.
{: .notice}  

<br />  

### 접근제어자  
main 함수에서 미리 보았지만 Java에는 접근제어자가 존재한다. 변수, 메서드 클래스에 대한 접근을 제한하는 기능을 한다. 총 4가지의 접근제어자가 존재하는데 public, protected, default, private 순으로 외부에 열려있다.  

- `public` : 모든 패키지의 모든 클래스에서 접근가능하다.  
- `protected` : 다른 패키지에서 접근시 상속을 통해서만 접근가능
- `default` : 동일 패키지에서만 접근 허용
- `private` : 동일 패키지 클래스 내부에서만 접근가능 



<br />  
