---  
layout: default
title: 1. SpringApplication 초기화 과정
last_modified_date: 2023-12-27 23:00:00
parent: Spring Boot Deep Dive
grand_parent: Spring
---  

1.Spring Boot 실행 Deep Dive - SpringApplication 초기화 과정
{: .head}

{: .no_toc }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>  

<br />  


Spring Boot 어플리케이션을 처음 생성 후 실행 시 메인함수의 `SpringApplication.run(메인클래스.class, args)` 메서드가 실행되며 스프링 어플리케이션이 동작하게 된다. 해당 메서드의 뒤에서 우리가 작성한 Bean들과 자동구성 Bean들이 인스턴스화되고, 내장 웹서버가 동작하는 등 Spring 어플리케이션 구동을 위한 모든 것들이 진행되게 된다.

이렇듯 Spring Boot는 어플리케이션으로 동작하기 위한 많은 부분들을 자동으로 만들고 처리해준다. 이것이 스프링의 장점이기도 하지만 실제로 어떤 코드들이 어떻게 구축되어 구동되는지가 궁금해서 직접 소스코드를 뒤적여 보았다.

main 메서드의 run 메서드가 실행된 이후에는 다음의 핵심 과정을 거쳐서 어플리케이션이 동작하게 된다. 
1. **SpringApplication 객체 생성**
2. **인스턴스 메서드 `run()` 실행(ApplicationContext 생성 및 설정이 일어난다)**
3. **`run()` 과정 중 `refresh()` 실행(컴포넌트 스캔 & 자동구성 빈 등록, 빈 객체화, 프록시 빈 생성)**
4. **Spring 구동 완료**

이번 글에서는 첫 번째 단계인 *'SpringApplication 객체가 생성되는 과정'* 에 대해서 정리해보았다.

{: .important}
> Spring Boot 3.2.1 버전을 기준으로 작성되었습니다.

<br />  

### SpringApplication 객체 생성
`SpringApplication.run(클래스, args)` 정적 메서드를 살펴보면 아래의 단계를 거치면서 `SpringApplication`객체를 생성하면서 인스턴스 메서드 run을 실행한다. 

재미있는 점은 run 메서드가 `ConfigurableApplicationContext` 타입을 리턴한다는 것이다. 해당 타입은 인터페이스로 `ApplicationContext` 가 가져야할 핵심적인 역할들을 지니고 있다. 후에 `ApplicationContext`의 상속관계를 알아볼 때 그 역할에 대해 자세하게 들여다보자. 

```java
/* SpringApplication.java */

// ...

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {  
   return run(new Class<?>[] { primarySource }, args);  
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {  
   return new SpringApplication(primarySources).run(args);  
}

```

SpringApplication의 생성자를 보면 **어플리케이션 타입추론, Initializer들 등록, Listener 등록, 메인클래스의 타입추론**이 이루어진다. Spring Boot 2.7.x 버전까지만 해도 `ApplicationContextFactory`와 `ApplicationStartup`에 대한 셋팅도 생성자에서 이루어졌지만 현재(3.2.1)는 선언과 동시에 초기화되도록 바뀌었다. 이제 생성자 코드를 하나씩 살펴보자.

{: .important}
> **ApplicationContextFactory**
> 
> `ApplicationContextFactory`는 `ApplicationContext` 객체를 생성하는 팩토리 클래스이다. `ApplicationStartup.DEFAULT`를 내부를 보면 구현체인 `DefaultApplicationContextFactory`를 사용하는 것을 볼 수 있다. 후에 `run` 인스턴스 메서드 실헹 시 사용된다.

{: .important}
> **ApplicationStartup**
> 
> `ApplicationStartup`은 진단, 측정 시간을 계측할 수 있게 도와주는 측정 도구이다. SpringApplication이 실행되는 과정을 여러 단계로 나누어서 측정하는데 사용된다. 실제로 코드 중간 중간 `ApplicationStartup`을 이용해서 태깅하는 것을 볼 수 있다.  

<br />  

```java
/* SpringApplication.java */

private ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;  
  
private ApplicationStartup applicationStartup = ApplicationStartup.DEFAULT;

// ...

public SpringApplication(Class<?>... primarySources) {  
   this(null, primarySources);  
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { 

   this.resourceLoader = resourceLoader;  
   Assert.notNull(primarySources, "PrimarySources must not be null");  
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));  

    // 어플리케이션 타입추론
    this.webApplicationType = WebApplicationType.deduceFromClasspath();  

    // BootStrapContext Initializer들 등록
    this.bootstrapRegistryInitializers = new ArrayList<>(  
         getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  

    // Application Initializer들 등록
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));  

    // ApplicationListener 등록
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));  
	
    // 메인클래스 Class 타입추론
    this.mainApplicationClass = deduceMainApplicationClass();  
}
```

<br />  

### 어플리케이션 타입추론
SpringApplication 은 총 3개의 `WebApplicationType`을 가지고 있다. 전통적인 MVC인 Servlet 기반 웹 어플리케이션, 반응형 이벤트 기반의 Reactive 웹 어플리케이션, 그리고 Spring Batch와 같이 웹 이외의 용도로 사용되는 어플리케이션.

`deduceFromClasspath`는 이중 어떤 Class를 가지고 있는지, 즉, **어떤 클래스를 가지고 있는지 확인하여 내 어플리케이션의 타입을 알려준다.** 
- Servlet을 사용하면 `DispatcherServlet` 클래스를 가지고 있으므로 `Servlet`
- Reactive를 사용하면 `DispatcherHandler` 클래스를 가지고 있으므로 `Reactive`
- Spring Boot가 아닌 Spring Framework를 독립적으로 사용하면 `None`

`WebApplicationType`이 무엇이냐에 따라 후에 `ApplicationContext` 구현체가 달라진다.

```java
/* SpringApplication.java */

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { 
	// ...
    this.webApplicationType = WebApplicationType.deduceFromClasspath();  
    // ...
}
```

```java
/* WebApplicationType.java */

public enum WebApplicationType {  
    NONE,  
    SERVLET,
    REACTIVE;  

    // 정적변수 - 클래스 Fully Qualified Name 들
  
   static WebApplicationType deduceFromClasspath() {  
      if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)  
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {  
         return WebApplicationType.REACTIVE;  
      }  
      for (String className : SERVLET_INDICATOR_CLASSES) {  
         if (!ClassUtils.isPresent(className, null)) {  
            return WebApplicationType.NONE;  
         }  
      }  
      return WebApplicationType.SERVLET;  
   }
}
```  

<br />  

### Initializer와 ApplicationListeners 등록
`bootstrapRegistryInitializers`는 `BootStrapContext`라는 `ApplicationContext`가 생성되기 전 부트스트랩 단계에서 필요한 임시 컨텍스트를 위한 것들이다. 등록되는 Initializer들은 `BootStrapContext`에 부가적으로 필요한 것들을 담당하는 객체들이다.
`BootStrapContext`는 `ApplicationContext`를 생성 전에 `SpringApplicationRunListener`(SpringApplication 라이프사이클 알림)를 처리하는 용도로 사용된다.

`setInitializers()` 메서드는 `ApplicationContext`에 부가적으로 필요한 것들을 담당하는 객체들이 등록된다. 마지막으로, `ApplicationListener`들도 등록된다.

```java
/* SpringApplication.java */

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { 

    // BootStrapContext Initializer들 등록
   this.bootstrapRegistryInitializers = new ArrayList<>(  
         getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  

    // Application Initializer들 등록
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));  

    // ApplicationListener 등록
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class)); 
   
   // ...
}

```

여기서 자세히 보면 세 과정 모두 `getSpringFactoriesInstances()` 메서드를 사용한다. 이름을 보면 인자로 넘겨받은 타입의 클래스들을 객체화해 불러오는 것으로 추측해볼 수 있다. 이 부분은 상당히 흥미로운데 의존된 외부 라이브러리들(External Libraries)의 jar 파일내부에 존재하는 `META-INF/spring.factories`들의 존재 의미를 알 수 있다.

<br />  

### SpringFactories의 정의된 클래스들 로드
`getSpringFactoriesInstances()` 메서드코드를 보면 두 단계로 나뉘어진다.
1. `SpringFactoriesLoader`의 정적메서드 `forDefaultResourceLocation`를 이용해서 `SpringFactoriesLoader` 클래스를 클래스화
2. `load` 메서드를 이용해 요청 타입의 객체들을 불러오기

```java
/* SpringApplication.java */
// ...
private <T> List<T> getSpringFactoriesInstances(Class<T> type) {  
   return getSpringFactoriesInstances(type, null);  
}  
  
private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {  

    // SpringFactoriesLoader 인스턴스화 후 load 
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);  
}
```

첫 번째 `forDefaultResourceLocation` 정적메서드는 ClassLoader와 타켓 공간을 설정해준다. 여기서 리소스를 불러올 타겟 공간을 `META-INF/spring.factories` 파일로 지정한다.

```java
/* SpringFactoriesLoader.java */

public class SpringFactoriesLoader {  
  
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
	
	public static SpringFactoriesLoader forDefaultResourceLocation(@Nullable ClassLoader classLoader) {  
		return forResourceLocation(FACTORIES_RESOURCE_LOCATION, classLoader);  
	}

	public static SpringFactoriesLoader forResourceLocation(String resourceLocation, @Nullable ClassLoader classLoader) {  
	   Assert.hasText(resourceLocation, "'resourceLocation' must not be empty");  
	   
	   ClassLoader resourceClassLoader = (
		   classLoader != null ? classLoader : SpringFactoriesLoader.class.getClassLoader());  
		 
	   Map<String, SpringFactoriesLoader> loaders = cache.computeIfAbsent(  
	         resourceClassLoader, key -> new ConcurrentReferenceHashMap<>());  

        // 타겟 공간 META-INF/spring.factories 로 초기화
	   return loaders.computeIfAbsent(resourceLocation, key ->  
	         new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)));  
	}

}
```

두번째 `load`  메서드는 넘겨준 타입에 맞는 클래스들의 인스턴스들을 넘겨받는다. load 메서드 로직의 핵심은 **넘겨받은 타입에 대한 구현체 클래스 목록을 불러오는 것**것과 이를 인스턴스화 하는 것이다.

```java
/* SpringFactoriesLoader.java */

public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver) {  
   return load(factoryType, argumentResolver, null);  
}

public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver,  
      @Nullable FailureHandler failureHandler) {  

    // 넘겨받은 타입을 구현하는 구현체 클래스 목록
    List<String> implementationNames = loadFactoryNames(factoryType);  

    List<T> result = new ArrayList<>(implementationNames.size());  

    FailureHandler failureHandlerToUse = (failureHandler != null) ? failureHandler : THROWING_FAILURE_HANDLER;  

    // 클래스들을 객체화
    for (String implementationName : implementationNames) {  
        T factory = instantiateFactory(implementationName, factoryType, argumentResolver, failureHandlerToUse);  
        if (factory != null) {  
            result.add(factory);  
        }  
    }  

    AnnotationAwareOrderComparator.sort(result);  
    return result;  
}
```

넘겨준 타입의 구현체 클래스들을 어떻게 찾을 수 있을까? 정답은 `META-INF/spring.factories` 이다. 각 라이브러리마다 해당 파일을 가지고 있는데 파일을 열어보면 아래와 같다. interface 명을 키로 그 구현체들을 값으로 가지는 파일이다.

```c
# Application Context Initializers  
org.springframework.context.ApplicationContextInitializer=\  
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\  
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\  
org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\  
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer  
  
# Application Listeners  
org.springframework.context.ApplicationListener=\  
org.springframework.boot.ClearCachesApplicationListener,\  
org.springframework.boot.builder.ParentContextCloserApplicationListener,\  
org.springframework.boot.context.FileEncodingApplicationListener,\  
org.springframework.boot.context.config.AnsiOutputApplicationListener,\  
org.springframework.boot.context.config.DelegatingApplicationListener,\  
org.springframework.boot.context.logging.LoggingApplicationListener,\  
org.springframework.boot.env.EnvironmentPostProcessorApplicationListener
```

`loadFactoryNames` 메서드는 모든 라이브러리들의 **`spring.factories` 파일을 확인하고 넘겨받은 타입을 키로 값에 해당되는 클래스 이름들을 모두 가지고 온다.** 

실제로 디버깅해보면 그 값들을 가지고 오는 것을 볼 수 있다. 아래 사진에서 SpringApplication 생성자에서 불러오는 `BootstrapRegistryInitializer`, `ApplicationContextInitializer`, `ApplicationListener` 구현체 클래스를 볼 수 있다. 재밌는 점은 3.2.1 버전기준으로 `BootstrapRegistryInitializer` 구현체는 존재하지 않는다.

*BootstrapRegistryInitializer*  
![BootstrapRegistryInitializer](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/8c8b6a8b-6685-45e8-9828-bf5bae9101af)  

*ApplicationContextInitializer*  
![ApplicationContextInitializer](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/08c84121-8a83-436e-ac59-e80e2436eb69)  

*ApplicationListener*  
![ApplicationListener](https://github.com/RokwonK/RokwonK.github.io/assets/52196792/a77077b2-622b-4666-8f50-d88285f8db60)  

<br />  

### 실행하는 메인클래스의 타입 추론
다시 SpringApplication 생성자로 돌아와서 마지막으로 메인클래스의 타입추론을 확인해보자. `mainApplicationClass`는 코드를 살펴보니 로그를 위해 셋팅하는 것 같다.(Docs를 읽어보니 로그를 위해 셋팅한다고 나와있다.)

구현체를 보니 `StackWalker` API의 `walk`메서드를 사용한다. `StackWalker`는 Java 9부터 도입된 클래스로 호출 스택(call stack)을 효울적으로 탐색, 조사할 수 있는 API이다. 필요한 콜 스택만을 불러올 수 있다는 장점을 지니고 있다.

`findMainCalss` 메서드를 보면 "main"가 있는 프레임을 찾아서 "main"함수가 선언된 클래스를 가져온다.

이제 SpringApplication이 생성되었다!

```java

/* SpringApplication.java */

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { 

	// 메인클래스 Class 타입추론
   this.mainApplicationClass = deduceMainApplicationClass();  
}

private Class<?> deduceMainApplicationClass() {  
   return StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE)  
      .walk(this::findMainClass)  
      .orElse(null);  
}

private Optional<Class<?>> findMainClass(Stream<StackFrame> stack) {  
   return stack.filter((frame) -> Objects.equals(frame.getMethodName(), "main"))  
      .findFirst()  
      .map(StackWalker.StackFrame::getDeclaringClass);  
}
```



### 정리
`SpringApplication`의 생성 과정에 대해서 알아보았다. 우리가 메인함수에서 `SpringApplication.run` 메서드를 실행하면 가장 먼저 `SpringApplication` 객체가 생성된다. 그리고 생성자에서 다음과 같은 일이 벌어진다.
1. **우리가 만든 웹어플리케이션의 타입을 지정한다.**
	- 어떤 클래스가 존재하는지 확인해 Servlet인지 Reactive인지 아닌지를 판별한다.
	- ApplicationContext을 생성할때 타입을 참고하여 각각 다른 구현체를 생성한다.
1. **`ApplicationContext`를 생성할 때 사용될 객체들을 셋팅한다.**
	- `ApplicationContextInitializers`와 `ApplicationListeners`를 셋팅하는 과정이다.
	- `META-INF/spring.factories` 파일에 정의된 내용을 토대로 구현체들을 찾고 생성한다.
2. **메인함수가 작성된 메인클래스를 확인하고 셋팅**
	- StackWalker API로 콜 스택을 확인하여 main 함수가 선언되어 있는 클래스를 불러온다.

<br />  

*'어떤 과정을 거쳐 Spring Container가 띄워지는가?'*라는 궁금증에서 시작해 코드를 살펴보았는데 잘 정리된 스프링 코드를 살펴보면서 코드 스타일에 대해 배울점이 많았을 뿐 아니라 잘 모르고 있었던 API들에 대해서도 알아가게 되는 것 같다.

이제 SpringApplication에서의 run 메서드에서 Spring Container가 띄워지는 과정을 알아볼 차례이지만 그 전에 Spring Container의 전신이라고 여겨지는 `ApplicationContext`를 먼저 살펴보자. 앞으로 이 시리즈는 `ApplicationContext`을 중심으로 흘러갈 예정일 것 같아 `ApplicationContext`를 먼저 알아보는게 흐름을 이해하는데 수월할 것 같다.