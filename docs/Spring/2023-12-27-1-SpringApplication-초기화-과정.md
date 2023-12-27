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
1. SpringApplication 객체 생성
2. 인스턴스 메서드 `run()` 실행(ApplicationContext 생성 및 설정이 일어난다)
3. `run()` 과정 중 `refresh()` 실행(컴포넌트 스캔 & 자동구성 빈 등록, 빈 객체화, 프록시 빈 생성)
4. Spring 구동 완료

이번 글에서는 그 첫번째 단계인 *'SpringApplication 객체가 생성되는 과정'* 에 대해서 정리해보았다.

{: .important }
> **Spring Boot 2.7.13을 기준으로 작성되었습니다.**  

<br />  

### SpringApplication 객체 생성
`SpringApplication.run(클래스, args)` 정적 메서드를 살펴보면 아래의 단계를 거치면서 `SpringApplication`객체를 생성하면서 인스턴스 메서드 run을 실행한다. 

재미있는 점은 run 메서드가 `ConfigurableApplicationContext` 타입을 리턴한다는 것이다. 해당 타입은 인터페이스로 `ApplicationContext` 가 가져야할 핵심적인 역할들을 지니고 있다. 후에 `ApplicationContext`의 상속관계를 알아볼 때 그 역할에 대해 자세하게 들여다보자.

```java
/* SpringApplication.java */

// ...

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {  
    return run(new Class[]{primarySource}, args);  
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {  
    return (new SpringApplication(primarySources)).run(args);  
}
```

SpringApplication의 생성자를 보면 상당히 많은 것들에 대한 설정이 이루어진다. 이 중 주석을 달아놓은 것들을 위주로 살펴보자. 

```java
/* SpringApplication.java */

// ...

public SpringApplication(Class<?>... primarySources) {  
    this((ResourceLoader)null, primarySources);  
}  
  
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {  
    this.sources = new LinkedHashSet();  
    this.bannerMode = Mode.CONSOLE;  
    this.logStartupInfo = true;  
    this.addCommandLineProperties = true;  
    this.addConversionService = true;  
    this.headless = true;  
    this.registerShutdownHook = true;  
    this.additionalProfiles = Collections.emptySet();  
    this.isCustomEnvironment = false;  
    this.lazyInitialization = false;  

	// ApplicationContext 생성객체
    this.applicationContextFactory = ApplicationContextFactory.DEFAULT;  
    
	// 기본 ApplicationStartup 설정
    this.applicationStartup = ApplicationStartup.DEFAULT;  
    
    this.resourceLoader = resourceLoader;  
    Assert.notNull(primarySources, "PrimarySources must not be null");  
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));  

	// 어플리케이션 타입추론
    this.webApplicationType = WebApplicationType.deduceFromClasspath();  

	// BootStrapContext Initializer들 등록
    this.bootstrapRegistryInitializers = new ArrayList(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  
    
    // Application Initializer들 등록
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));  

	// ApplicationListener 등록
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));  

	// 메인클래스 Class 타입추론
    this.mainApplicationClass = this.deduceMainApplicationClass();  
}
```

<br />  

### ApplicationContextFactory 설정
`ApplicationContextFactory`는 후에 `run` 인스턴스 메서드를 실행할때 ApplicationContext 객체를 생성하는 팩토리 클래스이다. `ApplicationStartup.DEFAULT`를 내부를 보면 `DefaultApplicationContextFactory`를 사용하는 것을 볼 수 있다. 

해당 클래스의 동작은 run 인스턴스 메서드를 알아볼 때 자세히 알아보자. 여기서는 **ApplicationContext를 생성하는 팩토리 클래스를 초기화**시킨다 까지만 알아두자.

```java
/* ApplicationContextFactory.java */

@FunctionalInterface  
public interface ApplicationContextFactory {  
    ApplicationContextFactory DEFAULT = new DefaultApplicationContextFactory();

	// ...
	
	ConfigurableApplicationContext create(WebApplicationType webApplicationType);

	// ...
}
```

<br />  

### ApplicationStartup
`ApplicationStartup`은 진단, 측정 시간을 계측할 수 있게 도와주는 측정 도구이다. SpringApplication이 실행되는 과정을 여러 단계로 나누어서 측정하는데 사용된다. 실제로 코드 중간 중간 `ApplicationStartup`을 이용해서 태깅하는 것을 볼 수 있다.

<br />  

### 어플리케이션 타입추론
SpringApplication 은 총 3개의 `WebApplicationType`을 가지고 있다. 전통적인 MVC인 Servlet 기반 웹 어플리케이션, 반응형 이벤트 기반의 Reactive 웹 어플리케이션, 그리고 Spring Batch와 같이 웹 이외의 용도로 사용되는 어플리케이션.

`deduceFromClasspath`는 이중 어떤 Class를 가지고 있는지(어떤 타입의 라이브러리를 사용하는지)를 확인하여 그 타입을 알려준다. `WebApplicationType`이 무엇이냐에 따라 후에 `ApplicationContext` 생성 시 생성되는 구현체가 달라진다.

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
  
    // ...
  
    static WebApplicationType deduceFromClasspath() {  
        if (ClassUtils.isPresent() /* 여러가지 조건식 */ ) {  
            return REACTIVE;  
        } else {  
            String[] var0 = SERVLET_INDICATOR_CLASSES;  
            int var1 = var0.length;  
  
            for(int var2 = 0; var2 < var1; ++var2) {  
                String className = var0[var2];  
                if (!ClassUtils.isPresent(className, (ClassLoader)null)) {  
                    return NONE;  
                }  
            }  
  
            return SERVLET;  
        }  
    }  
}
```

<br />  

### Initializer와 ApplicationListeners 등록
`bootstrapRegistryInitializers`는 `BootStrapContext`라는 `ApplicationContext`가 생성되기 전 부트스트랩 단계에서 필요한 임시 컨텍스트를 위한 것들이다. 등록되는 Initializer들은 `BootStrapContext`에 부가적으로 필요한 것들을 담당하는 객체들이다.
`BootStrapContext`는 `ApplicationContext`를 생성 전에 `SpringApplicationRunListener`(SpringApplication 라이프사이클 알림)를 처리하는 용도로 사용된다.

`setInitializers()` 메서드는 `ApplicationContext`에 부가적으로 필요한 것들을 담당하는 객체들이 등록된다. 마지막으로, `ApplicationListener`들도 등록된다.

여기서 세 과정 모두 `getSpringFactoriesInstances()` 메서드를 통해 해당되는 클래스들의 객체들을 불러오는 것을 볼 수 있다.

```java
/* SpringApplication.java */

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { 
	// ...
	
	// BootStrapContext Initializer들 등록
  this.bootstrapRegistryInitializers = new ArrayList(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class));  
    
  // Application Initializer들 등록
  this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));  

	// ApplicationListener 등록
  this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    
  // ...
}
```

<br />  

이렇게 SpringApplication가 생성되었다. 다시 run 메서드로 돌아가보면 생성 이후 바로 인스턴스 run 메서드를 호출하는 모습을 볼 수 있다. run메서드에서 Spring Container라고 불리는 `ApplicationContext`의 생성 및 실행된다.

이 과정을 살펴보기 전에 `ApplicationContext`가 어떻게 생겨먹었는지, 어떤 상속관계를 거쳐 어떤 역할을 맡고 있는지를 먼저 살펴보도록 하자.