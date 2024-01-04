---
title: "Spring Boot Deep Dive(4) - refresh (빈 구성 단계)"
excerpt: "컴포넌트 스캔과 자동구성 빈들이 실제로 등록되고 객체로 만들어지는 과정을 쫒아가보자"

categories:
  - Spring

permalink: /spring/spring-boot-deep-dive-4/
---

<br />  


main 함수에서 `SpringApplication`의 static 메서드 run을 실행하면 SpringApplication 객체가 필요한 데이터를 초기화하면서 생성된다. 이후에는 인스턴스 `run` 메서드를 실행하는데 환경변수 설정 및 실행환경인 `ApplicationContext`을 생성한다. `ApplicationContext`는 스프링 어플리케이션에서 Spring Container, DI Container라 불리며 스프링의 객체인 Bean에 대한 관리를 담당한다.

[이전 포스팅(Application 실행과정)](https://rokwonk.github.io/spring/spring-boot-deep-dive-3/)에서는 전체적으로 큰 흐름에 대해서 이야기 했다면, 이번 포스팅에서는 실제적으로 빈이 구성되는 단계인 refresh에 대해서 딥하게 들어가보자.

여담으로 이번 글은 *'Spring Boot Deep Dive 시리즈'* 를 포스팅하기 시작한 이유이다. **'컴포넌트 스캔과 자동구성은 대체 어디서 동작하는거지?'** 라는 궁금증에서 시작해서 하나씩 뜯어보다 드디어 기존의 궁금증을 해소할 수 있었다. 만약 필자와 같은 의문을 가졌던 분들이 계시다면 이번 포스팅을 통해 궁금증을 조금이나마 해소할 수 있으면 좋겠다.

> Spring Boot 3.2.1 버전을 기준으로 작성되었습니다.

<br />  

## Refresh 코드
`SpringApplication` 실행(run) 과정 중 afterContext 메서드는 `ApplicationContext`의 refresh() 메서드를 호출한다. [`SpringApplication`의 실행과정](https://rokwonk.github.io/spring/spring-boot-deep-dive-3/)과 [`ApplicaitonContext`의 상속관계](https://rokwonk.github.io/spring/spring-boot-deep-dive-2/)에 대한 자세한 내용은 각 링크에 작성해두었다.

```java
/* SpringApplication.java */
public ConfigurableApplicationContext run(String... args) {
	// ...
	try {
		// ...
		refreshContext(context);
	}
	// ...
}

private void refreshContext(ConfigurableApplicationContext context) {  
	// ...
	refresh(context);  
}

protected void refresh(ConfigurableApplicationContext applicationContext) {  
	// refresh 실행!
	applicationContext.refresh();  
}
```

refresh() 메서드는 상속구조 중 추상 클래스인 `AbstractApplicationContext`에 **템플릿 메서드 패턴의 형태로 구현**되어 있다. 일련의 단계가 순차적으로 실행되며 일부 메서드들은 서브 클래스에서 재정의하여 자체적인 동작을 추가한다. 주석과 함께 소스코드를 훑어보자면 아래와 같다.

```java
/* AbstractApplicationContext.java */

@Override  
public void refresh() throws BeansException, IllegalStateException {  
	this.startupShutdownLock.lock();  
	try {  
		// refresh 과정 동안만 StartupShutdown 쓰레드를 현재 쓰레드로 설정
		this.startupShutdownThread = Thread.currentThread();  
		
		StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");  
		
		// Refresh 준비
		prepareRefresh();  
		
		// Beanfactory 가져오기
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
		
		// BeanFactory 준비(BeanFactory 설정)
		prepareBeanFactory(beanFactory);  
	
		try {  
			// (서브 클래스 구현) 설정 완료 후 필요로직 호출
			postProcessBeanFactory(beanFactory);  
			
			StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");  
			
			// BeanFactoryPostProcessor들 실행
			invokeBeanFactoryPostProcessors(beanFactory);  
			
			// BeanPostProcessor 등록
			registerBeanPostProcessors(beanFactory);
			beanPostProcess.end();  
			
			// Initialize message source & ApplicationEventMulticaster
			initMessageSource();  
			initApplicationEventMulticaster();  
			
			// (서브 클래스 구현) 각 클래스에서 처리할 특별한 빈 구성
			onRefresh();  
			
			// Check for listener beans and register them.  
			registerListeners();  
			
			// 모든 빈이 생성된다.
			finishBeanFactoryInitialization(beanFactory);  
			
			// 마무리
			finishRefresh();
		}  
		catch (RuntimeException | Error ex ) {  
			// ...
		}  
		finally {  
			contextRefresh.end();  
		}  
	}  
	finally {  
		this.startupShutdownThread = null;  
		this.startupShutdownLock.unlock();  
	}  
}
```

여러가지 단계를 거치는데 여기서 빈팩토리와 빈 등록과 관련된 단계만 추려서 살펴보자. 
1. **prepareRefresh**
	- 본격적인 Refresh에 앞서 준비하는 단계
2. **prepareBeanFactory**
	- BeanFactory 환경 설정
3. **postProcessBeanFactory**
	- BeanFactory 준비완료 후 각 서브클래스 필요에 의해 재정의한 로직들 실행
4. **invokeBeanFactoryPostProcessors**
	- BeanFactoryPostProcessor 들 실행
	- 모든 빈의 정의를 등록하는 과정
5. **registerBeanPostProcessors**
	- BeanPostProcessor 들 등록
6. **onRefresh**
	- 빈 정의가 등록된 이후 각 클래스에서 필요에 따라 재정의한 로직들 실행
7. **finishBeanFactoryInitialization**
	- 등록된 빈들을 객체화하는 단계

<br />  

### Refresh 준비 단계
크게 살펴볼 코드는 없다. `ApplicationContext`가 현재 실행 중임을 의미하는 `active` 변수를 true로 변경한다. 이후 `initPropertySources`를 호출하는데 현재 이 메서드는 비어있다.

```java
/* AbstractApplicationContext.java */
protected void prepareRefresh() {  
	// Switch to active.  
	this.startupDate = System.currentTimeMillis();  
	this.closed.set(false);  
	this.active.set(true);  
	
	if (logger.isDebugEnabled()) {  
		if (logger.isTraceEnabled()) {  
			logger.trace("Refreshing " + this);  
		}  
		else {  
			logger.debug("Refreshing " + getDisplayName());  
		}  
	}  
	
	// Initialize any placeholder property sources in the context environment.  
	initPropertySources();  
	
	// ...
}
```

<br />  

### BeanFactory 설정
실제 기능을 담당하는 내부적인 `BeanFactory`에 필요한 정보를 설정하는 단계이다. 
1. **`ClassLoader`와 같은 필요한 빈 등록**
2. **`Aware` 인터페이스 타입의 의존성 주입을 무시하도록 설정**
3. **몇 타입들에 대한 구현체 빈을 현재 `ApplicationContext`로 등록**

`CloassLoader` 등은 이미 사용되고 있는 객체로 등록하고, 대부분의 `Aware` 인터페이스들을 무시하도록 설정한다. Aware 인터페이스는 이를 구현한 클래스들에게 특정 클래스가 객체화되면 메서드를 실행시켜주는 역할이다. 이를 통해 의존성을 주입받았는데 현재는 다른 주입방식들을 더 많이 사용하여 Ignore 설정을 해주는 듯하다.

`ResourceLoader`, `ApplicationEventPublisher`, `ApplicationContext` 타입에 대한 구현체를 직접 등록해준다. 개발을 하다보면 `ResourceLoader`를 사용할 때나 `ApplicationEventPublisher`을 사용할때 해당 타입으로 의존성을 주입받는데 이것들은 사실 `ApplicationContext`를 사용하는 것과 같다. [ApplicationContext 상속관계](https://rokwonk.github.io/spring/spring-boot-deep-dive-2/)에서 살펴본 것과 같이 `AbstractApplicationContext`는 `ResourceLoader`를 상속받고 있으며 `ApplicationContext`는 `ApplicationEventPublisher`를 받기 때문에 이런 설정이 가능하다.

```java
/* AbstractApplicationContext.java */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) { 
	// 필요한 빈들 직접 등록
	beanFactory.setBeanClassLoader(getClassLoader());  
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));  
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));  
	
	// Aware 인터페이스들 무시
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));  
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);  
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);  
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);  
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class); 
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);  
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);  
	beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);  
	
	// Dependency 직접 설정
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);  
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);  
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);  
	
	// ... 
}
```

<br />  

### BeanFactory 준비 이후 서브클래스 로직 실행
`postProcessBeanFactory` 메서드는 자식 클래스들에서 재정의되어있다. `ServletWebServerApplicationContext`에서 Web과 관련된 로직들이 실행된다. 유틸클래스에 값 셋팅 및 `Aware` 인터페이스에 대한 설정이 들어간다.

```java
/* ServletWebServerApplicationContext */

@Override  
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {  
	beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));  
	beanFactory.ignoreDependencyInterface(ServletContextAware.class);  
	registerWebApplicationScopes();  
}

private void registerWebApplicationScopes() {  
	ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(getBeanFactory());  
	WebApplicationContextUtils.registerWebApplicationScopes(getBeanFactory());  
	existingScopes.restore();  
}
```

<br />  

### 빈 정의 파싱 및 등록
`invokeBeanFactoryPostProcessors` 메서드에서 대부분의 빈들의 `BeanDefinition`이 등록된다. 고대하던 **컴포넌트 스캔 및 자동 구성 빈들이 등록되는 과정**이다. 뿐만 아니라 빈 정의를 조작하기도 하는데 예를 들어 `@Value` 어노테이션이 달린 필드에 환경설정 값을 넣어준다.

코드를 보면 아주 간단하다. 모든 `BeanFactoryPostProcessor`에 대한 실행을 `PostProcessorRegistrationDelegate`에 위임한다. 

```java
/* ServletWebServerApplicationContext */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {  
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());  
	
	// ...
}
```

`PostProcessorRegistrationDelegate` 의 정적 메서드 `invokeBeanFactoryPostProcessors`에서는 모든 `BeanFactoryPostProcessor`가 실행된다. 우선순위에 따라 순서대로 실행되는데 우선순위를 정리하자면 다음과 같다.
1. `BeanDefinitionRegistryPostProcessor` (`BeanFactoryPostProcessor` 상속받음)
	- (1순위) 이 중 `PriorityOrderd`를 구현한 클래스
	- (2순위) 그 다음 `Ordered`를 구현한 클래스
	- (3순위) 나머지
2. `BeanFactoryPostProcessor`
	- (4순위) 이 중 `PriorityOrderd`를 구현한 클래스
	- (5순위) 그 다음 `Ordered`를 구현한 클래스
	- (6순위) 나머지

아래 코드는 1순위(`BeanDefinitionRegistryPostProcessor`, `PriorityOrderd`를 구현한 클래스) 후처리기를 실행하는 코드이다. 

필자가 가장 궁금해했던 컴포넌트 스캔과 자동 구성 빈들의 `BeanDefinition`을 등록하는 일을 담당하는 클래스도 1순위로 처리되는 후처리기인 `ConfigurationClassPostProcessor`라는 클래스이다.

```java
/* PostProcessorRegistrationDelegate */
public static void invokeBeanFactoryPostProcessors(  
      ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
	
	Set<String> processedBeans = new HashSet<>();

	if (beanFactory instanceof BeanDefinitionRegistry registry) {  
		List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>(); 
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
		
		// ...
		
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false); 
		
		// 1. PriorityOrdered를 구현한 클래스(1순위)만 currentRegistryProcessors에 담기
		for (String ppName : postProcessorNames) {  
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {  
				// 2. ConfigurationClassPostProcessor가 추가됨
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));  
				processedBeans.add(ppName);  
			}  
		}
		
		sortPostProcessors(currentRegistryProcessors, beanFactory);  
		registryProcessors.addAll(currentRegistryProcessors);  
		
		// 3. ConfigurationClassPostProcessor가 실행됨
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry, beanFactory.getApplicationStartup());  
		
		currentRegistryProcessors.clear();  
		
		// 우선순위에 따라 나머지 BeanFactoryPostProcessor 실행
		// .. 
	}
}

private static void invokeBeanDefinitionRegistryPostProcessors(  
      Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry, ApplicationStartup applicationStartup) {  
	
	for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {  
		StartupStep postProcessBeanDefRegistry = applicationStartup.start("spring.context.beandef-registry.post-process")  
			.tag("postProcessor", postProcessor::toString);  
		// 4. 메서드 실행
		postProcessor.postProcessBeanDefinitionRegistry(registry);  
		postProcessBeanDefRegistry.end();  
	}  
}
```

`ConfigurationClassPostProcessor`의 `postProcessBeanDefinitionRegistry(registry)` 메서드를 살펴보자. 이 메서드에서는 처음에 메인어플리케이션을 구한 다음 후 다음 3단계로 모든 빈 정의를 등록한다.
1. parser 생성
2. parser를 이용한 메인 어플리케이션 파싱
3. reader를 생성하고 reader를 통해 파싱된 `ConfigurationClass`들을 `BeanDefinition`으로 만들어 로드시킨다.

```java
/* ConfigurationClassPostProcessor */

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {

	// configCandidates (mainApplication만 남음)
	// ...

	// 1. parser 생성
	ConfigurationClassParser parser = new ConfigurationClassParser(  
	      this.metadataReaderFactory, this.problemReporter, this.environment,  
	      this.resourceLoader, this.componentScanBeanNameGenerator, registry);  
	  
	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);  
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());  
	
	do {  
		StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");  
		
		// 2. parsing
		parser.parse(candidates);  
		parser.validate();  
		
		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());  
		configClasses.removeAll(alreadyParsed);  
		
		if (this.reader == null) {  
		  this.reader = new ConfigurationClassBeanDefinitionReader(  
				registry, this.sourceExtractor, this.resourceLoader, this.environment,  
				this.importBeanNameGenerator, parser.getImportRegistry());  
		}  
		
		// 3. reader를 통한 빈정의들을 로드시키기
		this.reader.loadBeanDefinitions(configClasses);  
		
		// ...
	}  
	while (!candidates.isEmpty());
}
```

여기서 parser 역할을 하는 `ConfigurationClassParser`를 좀 더 살펴보자. 객체를 생성하는 생성자를 보면 드디어 필자의 호기심을 풀어줄 귀한 존재가 보인다. 바로 `ComponentScanAnnotationParser`이다. 이름부터가 *'나 컴포넌트스캔 파서야~ '* 라고 존재감을 뿜뿜하고 있지 않은가.

유레카~ 소리지르고 싶지만 마음을 진정시키고 차분하게 조금 더 살펴보자. `ConfigurationClassParser`를 생성한 후에 `mainApplication`(configCandidates)를 인자로 `parse` 메서드를 실행한다.

`parse`메서드에서는 크게 두 가지 메서드가 실행된다. **컴포넌트 스캔을 통해 개발자가 정의한 빈 클래스를 긁는 `parse(metadata, beanName)`와 `AutoConfigurationImportSelector`를 통해 자동 구성 빈 클래스를 가져오는 `deferredImportSelectorHandler`가 실행**된다.

해당 메서드들의 내부는 상당히 복잡한데 어노테이션 `@Configuration`, `@Component`을 기준으로 클래스들을 가져오는 것이다.

```java
class ConfigurationClassParser {
	// ...
	
	private final DeferredImportSelectorHandler deferredImportSelectorHandler = new DeferredImportSelectorHandler();
	
	public ConfigurationClassParser(MetadataReaderFactory metadataReaderFactory, 
	ProblemReporter problemReporter, Environment environment, ResourceLoader resourceLoader,  
	  BeanNameGenerator componentScanBeanNameGenerator, BeanDefinitionRegistry registry) {  
	
		this.metadataReaderFactory = metadataReaderFactory;  
		this.problemReporter = problemReporter;  
		this.environment = environment;  
		this.resourceLoader = resourceLoader;  
		this.propertySourceRegistry = (this.environment instanceof ConfigurableEnvironment ce ?  
			 new PropertySourceRegistry(new PropertySourceProcessor(ce, this.resourceLoader)) : null);  
		this.registry = registry;  
		
		// 드디어 찾았다 요놈
		this.componentScanParser = new ComponentScanAnnotationParser(  
			 environment, resourceLoader, componentScanBeanNameGenerator, registry);  
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, resourceLoader);  
	}
	
	
	public void parse(Set<BeanDefinitionHolder> configCandidates) {  
		for (BeanDefinitionHolder holder : configCandidates) {  
			BeanDefinition bd = holder.getBeanDefinition();  
			try {  
				if (bd instanceof AnnotatedBeanDefinition annotatedBeanDef) {  
					// 내부적으로 componentScanParser 로 우리가 선언한 빈들을 등록
					parse(annotatedBeanDef.getMetadata(), holder.getBeanName());
				}  
				// ... 다른 pasre 방법
			}  
			catch (BeanDefinitionStoreException ex) {  
				throw ex;  
			}  
			catch (Throwable ex) {  
				throw new BeanDefinitionStoreException(  
			   "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);  
			}  
		}  
		
		// 얘는 AutoConfiguration을 처리하는 하는 친구
		this.deferredImportSelectorHandler.process();  
	}
}
```

`BeanFactoryPostProcesor`을 실행하는 이 과정을 짧게 정리하자면 다음과 같이 진행된다.

- 우선순위에 따라 후처리기들을 실행
	- 1순위로 `ConfigurationClassPostProcessor` 실행
		- `ConfigurationClassParser`를 생성하여 빈 클래스 parse
			- `ComponentScanAnnotationParser`를 통한 컴포넌트 스캔으로 빈들을 불러옴
			- `DeferredImportSelectorHandler`를 통해 `AutoConfigurationImportSelector`를 읽어와 AutoConfiguration 빈 클래스들을 불러옴
		- `ConfigurationClassBeanDefinitionReader`를 통한 파싱한 클래스들을`BeanDefinition`로 등록
	- (추가) 4순위로 `PropertySourcesPlaceholderConfigurer`가 실행된다.
		- `PropertySourcesPropertyResolver`를 이용하여 `@Value`에 환경설정 값들을 추가한다.

<br />  

### BeanPostProcessor 등록
`BeanPostProcessor`는 빈 생성 이후 초기화 전후로 진행할 로직들이 들어간다. 빈 생성은 refresh의 마지막 단계에서 이루어진다.

`BeanFactoryPostProcessor`를 실행할 때 사용했던 `PostProcessorRegistrationDelegate`에게 `BeanPostProcessor` 등록을 위임한다.

```java
/* AbstractApplicationContext */

protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {  
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);  
}
```

위 메서드 내부에서는 `PostProcessorRegistrationDelegate`의 메서드 `invokeBeanFactoryPostProcessors`와 마찬가지로 우선순위(PriorityOrdered 타입 - Ordered 타입 - 나머지 순)를 나누어 순위대로 `BeanPostProcessor`를 등록한다.

대표적으로 다음과 같은 `BeanPostProcessor`가 등록된다. 
- `AutowiredAnnotationBeanPostProcessor`
	- `@Autowired` 어노테이션을 처리하여 **의존성 주입처리**
- `PersistenceAnnotationBeanPostProcessor`
	- JPA 사용 시 적용되는 것으로 **EntityManager 주입**을 위한 `@PersistenceContext` 어노테이션 처리 
- `ConfigurationPropertiesBeanPostProcessor`
	- `@ConfigurationProperties` 어노테이션을 처리하여 `PropertySource` 바인딩
- `AnnotationAwareAspectJAutoProxyCreator`
	- `@AspectJ`뿐만 아니라 `@Aspect`도 처리하여 **프록시 빈을 생성**
	- 내부적으로 `ProxyFactory`를 통해 `ObjenesisCglibAopProxy`, `JdkDynamicAopProxy`를 이용해 프록시 객체를 만들어 낸다.

<br />  

### 빈 정의 등록 후 서브클래스 로직 실행
`onRefresh` 메서드는 서브클래스인 `ServletWebServerApplicationContext`에 재정의 되어 있다. `createWebServer()`라는 이름답게 웹서버를 생성하는 메서드이다. 해당 메서드 내부에서는 `getWebServer()`로 Tomcat등의 설정한 웹서버가 만들지면서 동시에 `ServletContext`(서블릿 실행환경)도 띄워진다. 이때 `ServletContextInitializer`라는 콜백 함수를 넘겨서 `ServletContext`를 생성되면 멤버 변수에 `ServletContext`를 저장한다.(해당 변수는 `ServletWebServerApplicationContext`의 부모 클래스인 `GenericWebApplicationContext`에 존재한다.)

```java
/* ServletWebServerApplicationContext.java */

@Override  
protected void onRefresh() {  
	super.onRefresh();  
	try {  
		createWebServer();  
	}  
	catch (Throwable ex) {  
		throw new ApplicationContextException("Unable to start web server", ex);  
	}  
}

private void createWebServer() {  
	WebServer webServer = this.webServer;  
	ServletContext servletContext = getServletContext();  
	if (webServer == null && servletContext == null) {  
	
		ServletWebServerFactory factory = getWebServerFactory();  
		
		// WeberServer 생성 및 콜백함수로 ServletContext 생성
		this.webServer = factory.getWebServer(getSelfInitializer());  
		
		getBeanFactory().registerSingleton("webServerGracefulShutdown",  
			new WebServerGracefulShutdownLifecycle(this.webServer));  
		getBeanFactory().registerSingleton("webServerStartStop",  
			new WebServerStartStopLifecycle(this, this.webServer));  
	}  
	else if (servletContext != null) {  
		try {  
			getSelfInitializer().onStartup(servletContext);  
		}  
		catch (ServletException ex) {  
			throw new ApplicationContextException("Cannot initialize servlet context", ex);  
		}  
	}  
	initPropertySources();  
}

private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {  
	return this::selfInitialize;  
}  
  
private void selfInitialize(ServletContext servletContext) throws ServletException {
	// 내부 멤버 변수에 servletContext 저장
	prepareWebApplicationContext(servletContext);  
	registerApplicationScope(servletContext);  
	WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);  
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {  
		beans.onStartup(servletContext);  
	}  
}

```

<br />  

### 빈 객체 생성
등록된 `BeanDefinition`을 이용해서 실제로 빈들을 생성하는 단계이다. 빈들이 생성될 뿐만 아니라 등록한`BeanPostProcessor`들을 실행하면서 필요한 어노테이션을 처리하고 의존성을 주입하며, 필요에 의해 프록시 빈들(@Trasactional, @Aspect 어노테이션 등이 달린 클래스 및 메서드들)도 생성된다.

```java
/* ServletWebServerApplicationContext.java */

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {  
	
	// ...
	
	beanFactory.preInstantiateSingletons();  
}
```

내부적으로 사용하는 `DefaultListableBeanFactory`의 `preInstantiateSingletons()`를 통해 이러한 과정들이 전부 실행된다. 결론적으로 `DefaultListableBeanFactory`가 상속 받는 클래스인 `AbstractAutowireCapableBeanFactory`의 `createBean` 메서드를 통해 빈 생성 및 `BeanFactoryPostProcessor`를 통한 의존성 주입, 프록시 빈 들이 만들어진다.

이 과정이 마무리되고 나서 사용했던 자원, 캐시들을 반납하며 refresh 과정을 마무리한다. 이후에는 [SpringApplication 실행(run 과정)](https://rokwonk.github.io/spring/spring-boot-deep-dive-3/)에서 이야기한 Runner들을 실행하고 유저들의 요청을 받을 수 있는 상태가 된다.

<br />  

### 마무리
지금까지 스프링 부트 프로젝트가 어떤 과정을 거치며 실행되는지에 대해 큰 흐름을 기준으로 살펴보았다. 여러 과정을 거치면서 기존의 의문점들이 많이 풀렸을 뿐만 아니라 단계마다 사용되는 각 객체들이 어떤 역할을 하는지 찾아보면서 새롭게 알게된 기능이나 API들도 여럿 있었다.

생각 이상으로 배운 것들이 많아 앞으로도 개발을 하면서 동작과정에 궁금증이 생긴다면 Deep Dive 시리즈로 포스팅 해보고자 한다.

