---
title: "Spring boot의 자동구성"
categories: Spring
tags:
  - Annotation
---  

# Spring boot의 자동화

스프링 부트는 개발자가 수동으로 작성해야하는 코드를 최소화 시켜주고 많은 기능들을 자동화 해준다. 그리고 이러한 자동화는 Annotation(이하 어노테이션)을 기반으로 이루어진다. 

## Annotation
스프링에서 어노테이션이란 프로그램에게 **추가적인 정보를 제공해주는 메타데이터**이다. 

컴파일러에게 문접 에러 체크를 위한 정보를 제공, 코드를 자동으로 생성할 수 있도록 정보 제공, 런타임에 특정 기능을 실행하도록 정보를 제공하는 등의 역할을 한다. 스프링에서 제공하는 어노테이션들은 런타임에 **특정 기능을 실행하도록 정보를 제공하는 역할**이 대부분이다.

어노테이션을 이용함으로써 얻을 수 있는 장점은 다음과 같다.

- 코드의 가독성 향상
- 코드의 반복이 줄어듬
- 비즈니스 로직과 다른 부가적인 기능, 내부 동작 코드를 분리  

<br />

### 어노테이션 정의하기

`@interface` 를 이용하여 어노테이션을 직접 구현할 수 있다.

```java
@Target({ElementType.적용대상})
@Retention(RetentionPolicy.유지정책)
@Documented
public @interface 어노테이션이름 {

}
```

`@Target` 은 어노테이션을 적용할 대상을 선택한다.생성자, 필드, 메서드 등 선언할 수 있는 대상을 선택한다.

`@Retension` 은 어노테이션이 유지될 시간을 정한다. Class(클래스 파일에 기록되지만 런타임에는 유지되지 않는다), RUNTIME(런타임에도 유지), SOURCE(소스에만 반영되고 컴파일러에 의해 삭제)

`@Documented` 는 javadoc(클래스 설명 문서)에 자동으로 기록된다.(필수X)

`@Inherited` 는 이 어노테이션을 가진 클래스를 상속받는 클래스들에게도 어노테이션을 적용시킨다.(필수X)

`@interface` 는 어노테이션 선언 클래스임을 나타낸다.  

<br />

### 어노테이션 활용하기 - Reflection

**리플렉션은 클래스와 메서드, 필드의 정보를 동적으로 가져오는 기능**이다. 이를 이용하여 런타임 중에 클래스, 메서드, 필드에 부여된 어노테이션(메타데이터)을 확인하고 추가적인 행동을 처리할 수도 있다. 또한 객체를 생성하고 메서드 호출, 필드 액세스 등을 조작할 수 있지만 너무 많이 사용하면 성능 저하가 발생할 수 있다.

다음은 스프링이 Reflection으로 메타정보인 어노테이션을 이용하는 예시이다.

- 스프링의 DI의 원리 - Component Scan 시 클래스 스캔방식으로  `@Component`, `@Bean` 어노테이션을 가진 클래스를 찾고 → 싱글톤으로 생성 → 해당 객체를 의존하는 클래스의 `@Autowired` 어노테이션에 주입하여 객체를 생성하는 방법을 반복한다.
- JPA의 데이터베이스 테이블과 Entity 객체간 매핑 - `@Entity` 어노테이션을 가진 클래스의 필드를 조사하여 실제 데이터베이스의 테이블, 컬럼과 매핑시켜 검증 또는 테이블을 생성한다.  

이렇게 어노테이션과 리플렉션의 조합으로 코드의 간결함과 가독성, 확장성을 얻을 수 있다. 아래는 리플렉션을 이용하여 클래스 정보를 가져오는 여러가지 예시이다.

```java
public class TestReflection {

  public static void main(String[] args) throws Exception {
    // 인스턴스 객체 이용하기
    // <?> : 어떤 종류의 클래스든 다 얻어올 수 있다는 의미
    String obj = new String("Reflection Test");
    Class<?> c1 = obj.getClass(); 
    
    // Class.forName()
    // 문자열이 가리키는 곳에 클래스가 없을 수도 있으므로 예외처리가 필요
    Class<?> c2 = Class.forName("java.lang.String"); 
        
    // class 정적 변수(모든 클래스에 존재한다.)
    // 클래스명을 직접적으로 사용하므로 유지보수가 어려워짐
    Class<?> c3 = String.class;
  }
}
```

<br />

## Spring Boot의 구성구성 도와주는 어노테이션

우리가 스프링 부트 프로젝트를 만들면 가장 먼저 `@SpringBootApplication` 어노테이션이 붙은 클래스이다. 이 어노테이션은 스프링 컨테이너를 구성하는 역할을 한다. 어노테이션을 까보면  `@SpringBootConfiguration` , `@EnableAutoConfiguration` , `@ComponentScan`  로 이루어져 있는 것을 알 수 있는데 이 중 `@EnableAutoConfiguration` , `@ComponentScan` 가 스프링의 빈들을 자동구성하는 주요한 역할을 한다.

### @SpringBootConfiguration

이 어노테이션은 의미론적인 어노테이션이다. 스프링 부트 어플리케이션의 구성 클래스임을 나타내고 자동 구성을 활성화하는 역할을 수행한다는 의미를 지닌다.

### @ComponentScan

개발자가 직접 작성한 `@Repository` , `@Service` , `@Controller` , `@Component` , `@Configuration`, `@Bean` 등을 스프링 컨텍스트에 빈으로 등록시키는 역할을 한다.

### @EnableAutoConfiguration

ComponentScan으로 1차적으로 빈을 등록한 후 2차로 빈을 등록하는 과정을 거친다. 이 어노테이션은 어플리케이션 동작에 **필요한 빈(라이브러리와 관련된 등)을 자동으로 구성하는 기능을 담당한다**. 여기서 필요한 빈이란 모듈, 라이브러리들과 관련된 객체들이다. 덕분에 라이브러리와 연동할때 필요한 객체들을 빈에 직접 등록하지 않아도 자동으로 필요한 빈들을 구성해준다.(데이터베이스 Driver, JPARepository 인터페이스를 구현한 객체 등)

spring boot 2.7 이상부터는 자동 구성의 대상이 되는 후보 빈들은 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 에 등록되어 있다. 이전 버전들에서는 `META-INF/spring.factories` 내부 EnableAutoConfiguration 섹션에 등록되어 있다.  

<br />  

**참고**  
- [https://velog.io/@gale4739/Spring-Boot-어노테이션](https://velog.io/@gale4739/Spring-Boot-%EC%96%B4%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98)
- https://brunch.co.kr/@anonymdevoo/49
- [https://velog.io/@suyeon-jin/리플렉션-스프링의-DI는-어떻게-동작하는걸까](https://velog.io/@suyeon-jin/%EB%A6%AC%ED%94%8C%EB%A0%89%EC%85%98-%EC%8A%A4%ED%94%84%EB%A7%81%EC%9D%98-DI%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EB%8F%99%EC%9E%91%ED%95%98%EB%8A%94%EA%B1%B8%EA%B9%8C)