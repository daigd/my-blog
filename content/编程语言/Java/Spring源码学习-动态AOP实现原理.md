---
title: "Spring源码学习-动态AOP实现原理"
date: 2020-10-30T11:09:12+08:00
tags: ["Spring", "源码学习","AOP"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

IOC 和 AOP 是Spring的两大核心功能，接下来关注AOP的实现，首先来看下使用案例。

[源码Github地址](https://github.com/daigd/StudyDemo/tree/master/spring-source-learning)。

先引入对应依赖，如下，spring.version 为实际版本号：

```xml
<!--bean,context依赖 begin-->
 <dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-context</artifactId>
     <version>${spring.version}</version>
 </dependency>
 <!--bean,context依赖 end-->
 
 <!--aop依赖 begin-->
 <dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-aop</artifactId>
     <version>${spring.version}</version>
 </dependency>
 <dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-aspects</artifactId>
     <version>${spring.version}</version>
 </dependency>
 <!--aop依赖 end-->

 <!--单元测试依赖 begin-->
 <dependency>
     <groupId>org.junit.jupiter</groupId>
     <artifactId>junit-jupiter</artifactId>
     <version>5.6.2</version>
     <scope>test</scope>
 </dependency>
 <!--单元测试依赖 end-->
```

测试代码入口：

```java
public class MyAopTests
{
    @Test
    @DisplayName("aop验证,对get方法进行了增强")
    void xmlApplicationContextTest()
    {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("MyAopTests.xml");
        MyTestBean bean = (MyTestBean) ctx.getBean("myTestBean");
        Assertions.assertEquals("myTest", bean.getName());
    }
}
```

切面 Bean配置：

```java
@Aspect
public class AspectJTest
{
    @Pointcut("execution(* *.get*(..))")
    public void test()
    {
        
    }

    @Before("test()")
    public void beforeTest()
    {
        System.out.println("Before Test.");
    }

    @After("test()")
    public void afterTest()
    {
        System.out.println("After Test.");
    }
    
    @Around("test()")
    public Object aroundTest(ProceedingJoinPoint p){
        System.out.println("Before around.");
        Object obj = null;
        try
        {
            obj = p.proceed();
        }
        catch (Throwable throwable)
        {
            throwable.printStackTrace();
        }
        System.out.println("After around.");
        return obj;
    }
}
```

测试 Bean：

```java
public class MyTestBean
{
    private String name = "myTest";

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }
}
```

配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   default-lazy-init="false"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
		http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.1.xsd">

	<!--开启AOP功能-->
	<aop:aspectj-autoproxy/>
	<bean id="myTestBean" class="com.dgd.spring.beans.factory.MyTestBean"/>
	<bean class="com.dgd.spring.aop.AspectJTest"/>

</beans>
```

执行测试用例，不出意外，控制台输出如下：

```bash
Before around.
Before Test.
After around.
After Test.
```

下面来看下Spring AOP 逻辑是如何实现的。

## 解析标签

因为 `<aop:aspectj-autoproxy/>` 不属于 Spring 的默认标签，因为这里涉及到自定义标签的解析，所以 Spring 一定在某个地方注册了对应的解析器，全局搜索 aspectj-autoproxy ，不难找到在 AopNamespaceHandler 类里，对应着下面这行代码：

```java
registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
```

因此，AspectJAutoProxyBeanDefinitionParser 就是 AOP 标签的解析器，我们看下其 parse() 的实现：

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
	// 注册 AnnotationAwareAspectJAutoProxyCreator
	AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
	// 对扩展元素的处理
	extendBeanDefinition(element, parserContext);
	return null;
}
```

AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary() 是我们比较关心的，也是核心逻辑实现：

```java
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
		ParserContext parserContext, Element sourceElement) {

	// 注册代理 beanDefinition
	BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
			parserContext.getRegistry(), parserContext.extractSource(sourceElement));
    // 处理 proxy-target-class、expose-proxy 属性
	useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
	registerComponentIfNecessary(beanDefinition, parserContext);
}
```

### 注册代理 beanDefinition

顺着代码入口看下去：

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
		BeanDefinitionRegistry registry, @Nullable Object source) {

	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

private static BeanDefinition registerOrEscalateApcAsRequired(
		Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

	if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
		// 如果注册了创建代理 beanDefinition,比较 className 跟 beanName 是否一致,如果一致直接返回null,不做后续处理；
		// 否则按优先级来决定是否需要重设 className
		BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
		if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
			int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
			int requiredPriority = findPriorityForClass(cls);
			if (currentPriority < requiredPriority) {
				apcDefinition.setBeanClassName(cls.getName());
			}
		}
		return null;
	}

	// 找不到代理 beanDefinition，则直接创建并注册
	RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
	beanDefinition.setSource(source);
	beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
	beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
	return beanDefinition;
}
```

因此，如果声明了 AOP 标签`<aop:aspectj-autoproxy/>`，解析器里就是注册代码 beanDefinition。

### 处理 proxy-target-class 和 expose-proxy 属性

代码 beanDefinition 注册完毕后，处理 proxy-target-class 和 expose-proxy 属性，

useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement)：

```java
private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, @Nullable Element sourceElement) {
	if (sourceElement != null) {
		// 获取 proxy-target-class、expose-proxy 属性值
		// 如果设置了则在代理 beanDefinition 添加属性 proxyTargetClass=true，exposeProxy=true
		boolean proxyTargetClass = Boolean.parseBoolean(sourceElement.getAttribute(PROXY_TARGET_CLASS_ATTRIBUTE));
		if (proxyTargetClass) {
			AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
		}
		boolean exposeProxy = Boolean.parseBoolean(sourceElement.getAttribute(EXPOSE_PROXY_ATTRIBUTE));
		if (exposeProxy) {
			AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
		}
	}
}
// 强制使用也是一个设置属性过程
public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
	if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
		BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
		definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
	}
}
```

下面来总结下 proxyTargetClass 和 exposeProxy作用：

- proxyTargetClass ：强制使用 CGLIB 代理

Spring AOP 动态代理有两种方式，分别是 JDK 动态代理和 使用 CGLIB 代理，如果代理目标至少实现了一个目标，则使用 JDK 动态代理，如果目标对象没有实现任何接口，则创建 CGLIB 代理，如果配置文件指定 proxy-target-class=true，则强制使用 CGLIB 代理。

- exposeProxy：解决同一个类方法相互调用切面不生效的问题。

## 创建 AOP 代理

通过解析自定义标签完成 AOP beanDefinition 的创建之后，后续 AOP 代理的创建就可以由其实现了。由解析标签代码可知，创建的 beanDefinition 类型是 AnnotationAwareAspectJAutoProxyCreator，这个类干啥用的？看下它的类图：

![](/java/img/AnnotationAwareAspectJAutoProxyCreator.png)

可知其实现了 BeanPostProcessor 接口，因为 Spring 在加载 Bean实例化前会调用 postProcessAfterInitialization() 方法，由此作为分析入口，源代码在 AbstractAutoProxyCreator.postProcessAfterInitialization() 中，如下：

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
	if (bean != null) {
		// 获取缓存 key, 如果 beanName 不为空返回 beanName，如果是 FactoryBean 类型，beanName 前加 &
		// beanName 为空，返回 beanClass 
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		// 代理集合,未创建代理的话返回为null
		if (this.earlyProxyReferences.remove(cacheKey) != bean) {
			// 如果适合被代理，做一些额外工作
			return wrapIfNecessary(bean, beanName, cacheKey);
		}
	}
	return bean;
}
```

重点关注 wrapIfNecessary(bean, beanName, cacheKey) 方法，实现如下：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	// 对应beanName已有被包装后的代理,直接返回
	if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	// advisedBeans 集合里标识指定 bean 是否需要增强
	// 如果不是增强 bean,则指定bean无需进行包装处理,直接返回
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	// 如果是基础设施 bean,或指定跳过包装阶段,直接返回,并将该 bean 标识成无需增强,保存在 advisedBeans 集合中
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// Create proxy if we have advice.
	// 获取 AOP 增强器,确定要给代理 bean 添加哪些功能
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 创建对应 bean 代理
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		// 保存代理类型
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}
	// 如果没有找到匹配的增强器,则不需要创建代理,标识对应 bean 不需要增强 
	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
```

除了一些判断逻辑之外，wrapIfNecessary(bean, beanName, cacheKey) 方法主要做了两件事：

- 获取特定拦截器（这里是封装好的 AOP 通知，或所谓增强器）；
- 根据增强器创建代理。

### 获取切面增强器

来看下获取增强器 getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null) 的实现逻辑：

```java
protected Object[] getAdvicesAndAdvisorsForBean(
		Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
	// 找到合格的增强器
	List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
	if (advisors.isEmpty()) {
		return DO_NOT_PROXY;
	}
	return advisors.toArray();
}

protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	// 找到候选的增强器
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	// 从候选通知器中挑选出可以应用在当前代理类上的（通过切点的表达式来判断）
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	// 对增强器预留的扩展（默认空实现）
    extendAdvisors(eligibleAdvisors);
	if (!eligibleAdvisors.isEmpty()) {
        // 对挑选出来的增强器进行排序
		eligibleAdvisors = sortAdvisors(eligibleAdvisors);
	}
	return eligibleAdvisors;
}
```

#### 找到候选增强器

通过 findCandidateAdvisors() 找到候选增强器，由于是通过注解方式获取切面通知，对应实现类是 AnnotationAwareAspectJAutoProxyCreator，对应实现：

```java
protected List<Advisor> findCandidateAdvisors() {
	// Add all the Spring advisors found according to superclass rules.
	// 通过父类获取到增强器（通过XML配置）
	List<Advisor> advisors = super.findCandidateAdvisors();
	// Build Advisors for all AspectJ aspects in the bean factory.
	if (this.aspectJAdvisorsBuilder != null) {
		// 通过注解方式定义的增强器
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
	}
	return advisors;
}
```

 AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors()通过继承父类实现，会同时获取到通过 XML 配置的切面通知，通过注解方式获取的逻辑：

```java
public List<Advisor> buildAspectJAdvisors() {
	// 构建切面增强器
	List<String> aspectNames = this.aspectBeanNames;
	if (aspectNames == null) {
		synchronized (this) {
			aspectNames = this.aspectBeanNames;
			if (aspectNames == null) {
				List<Advisor> advisors = new ArrayList<>();
				aspectNames = new ArrayList<>();
				// 获取所有注册的 bean
				String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Object.class, true, false);
				for (String beanName : beanNames) {
					if (!isEligibleBean(beanName)) {
						continue;
					}
					// We must be careful not to instantiate beans eagerly as in this case they
					// would be cached by the Spring container but would not have been weaved.
					Class<?> beanType = this.beanFactory.getType(beanName);
					if (beanType == null) {
						continue;
					}
					// 判断对应 bean 是否是切面类(是否声明@Aspect注解)
					if (this.advisorFactory.isAspect(beanType)) {
						aspectNames.add(beanName);
						AspectMetadata amd = new AspectMetadata(beanType, beanName);
						if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
							MetadataAwareAspectInstanceFactory factory =
									new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
							// 获取切面类的切面通知,封装成增强器 Advisor
							List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
							if (this.beanFactory.isSingleton(beanName)) {
								this.advisorsCache.put(beanName, classAdvisors);
							}
							else {
								this.aspectFactoryCache.put(beanName, factory);
							}
							advisors.addAll(classAdvisors);
						}
						else {
							// Per target or per this.
							if (this.beanFactory.isSingleton(beanName)) {
								throw new IllegalArgumentException("Bean with name '" + beanName +
										"' is a singleton, but aspect instantiation model is not singleton");
							}
							MetadataAwareAspectInstanceFactory factory =
									new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
							this.aspectFactoryCache.put(beanName, factory);
							advisors.addAll(this.advisorFactory.getAdvisors(factory));
						}
					}
				}
				this.aspectBeanNames = aspectNames;
				return advisors;
			}
		}
	}

	if (aspectNames.isEmpty()) {
		return Collections.emptyList();
	}
	// 保存在缓存：key-切面类的 beanName,value-切面类的增强器列表
	List<Advisor> advisors = new ArrayList<>();
	for (String aspectName : aspectNames) {
		List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
		if (cachedAdvisors != null) {
			advisors.addAll(cachedAdvisors);
		}
		else {
			MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
			advisors.addAll(this.advisorFactory.getAdvisors(factory));
		}
	}
	return advisors;
}
```

核心逻辑就是遍历所有注册的 bean,找到对应切面类，从切面类中找到切面通知，封装成增强器 Advisor，获取封装通知器的逻辑在advisorFactory.getAdvisors(factory) 中，实现如下：

```java
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
	// 切面类类型
	Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
	// 切面名称
	String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
	// 切面类校验
	validate(aspectClass);

	// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
	// so that it will only instantiate once.
	MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
			new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

	List<Advisor> advisors = new ArrayList<>();
	for (Method method : getAdvisorMethods(aspectClass)) {
		// 根据获取到的切面方法,封装成增强器返回
		Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
		if (advisor != null) {
			advisors.add(advisor);
		}
	}

	// If it's a per target aspect, emit the dummy instantiating aspect.
	// 如果配置了延迟初始化，添加一个同步实例化增强器，保证增强器在使用前都能初始化
	if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
		Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
		advisors.add(0, instantiationAdvisor);
	}

	// Find introduction fields.
	for (Field field : aspectClass.getDeclaredFields()) {
		// 获取 DeclareParents 注解配置信息
		Advisor advisor = getDeclareParentsAdvisor(field);
		if (advisor != null) {
			advisors.add(advisor);
		}
	}

	return advisors;
}
```

从切面类中获取通知器，主要逻辑：

- 切面类的校验；
- 遍历切面方法，封装成增强器返回；
- 切面通知如果配置了延迟初始化，添加一个同步实例化增强器；
- 将 DeclareParents 注解相关信息封装成增强器。

重点关注普通切面通知的获取及封装成增强器相关逻辑，获取切面方法,代码入口为 getAdvisorMethods(aspectClass)，具体实现：

```java
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
	final List<Method> methods = new ArrayList<>();
	ReflectionUtils.doWithMethods(aspectClass, method -> {
		// Exclude pointcuts
		// 排除声明了切点(@Pointcut)的方法 
		if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
			methods.add(method);
		}
	}, ReflectionUtils.USER_DECLARED_METHODS);
	if (methods.size() > 1) {
		// 对切面方法排序,按 Around, Before, After, AfterReturning, AfterThrowing 顺序排序
		methods.sort(METHOD_COMPARATOR);
	}
	return methods;
}
```

从获取的切面方法中提取增强器，Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName) ，实现：

```java
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
		int declarationOrderInAspect, String aspectName) {
	// 切面类校验
	validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
	// 获取切点
	AspectJExpressionPointcut expressionPointcut = getPointcut(
			candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
	if (expressionPointcut == null) {
		return null;
	}
	// 将切面方法及切点相关信息封装在 InstantiationModelAwarePointcutAdvisorImpl 类中
	return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
			this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

切面方法封装逻辑：

```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
		Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
		MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

	this.declaredPointcut = declaredPointcut;
	this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
	this.methodName = aspectJAdviceMethod.getName();
	this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
	this.aspectJAdviceMethod = aspectJAdviceMethod;
	this.aspectJAdvisorFactory = aspectJAdvisorFactory;
	this.aspectInstanceFactory = aspectInstanceFactory;
	this.declarationOrder = declarationOrder;
	this.aspectName = aspectName;

	// 针对延迟初始化的处理
	if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
		// Static part of the pointcut is a lazy type.
		Pointcut preInstantiationPointcut = Pointcuts.union(
				aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

		// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
		// If it's not a dynamic pointcut, it may be optimized out
		// by the Spring AOP infrastructure after the first evaluation.
		this.pointcut = new PerTargetInstantiationModelPointcut(
				this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
		this.lazy = true;
	}
	else {
		// A singleton aspect.
		this.pointcut = this.declaredPointcut;
		this.lazy = false;
		// 增强器实例化
		this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
	}
}
```

针对不同切面注解的实例化，委托给 instantiateAdvice(this.declaredPointcut) 来处理，如下：

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
	// 根据不同的切面注解获取对应通知
	Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
			this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
	return (advice != null ? advice : EMPTY_ADVICE);
}

public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
		MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

	Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
	// 切面类校验
	validate(candidateAspectClass);

	//从切面方法上获取对应注解: Pointcut, Around, Before, After, AfterReturning, AfterThrowing
	AspectJAnnotation<?> aspectJAnnotation =
			AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
	if (aspectJAnnotation == null) {
		return null;
	}

	// If we get here, we know we have an AspectJ method.
	// Check that it's an AspectJ-annotated class
	// 校验切面类是否是 AspectJ 类型，不是直接抛出异常
	if (!isAspect(candidateAspectClass)) {
		throw new AopConfigException("Advice must be declared inside an aspect type: " +
				"Offending method '" + candidateAdviceMethod + "' in class [" +
				candidateAspectClass.getName() + "]");
	}

	if (logger.isDebugEnabled()) {
		logger.debug("Found AspectJ method: " + candidateAdviceMethod);
	}

	AbstractAspectJAdvice springAdvice;

	// 针对不同注解返回不同通知,如果是切点返回null
	switch (aspectJAnnotation.getAnnotationType()) {
		case AtPointcut:
			if (logger.isDebugEnabled()) {
				logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
			}
			return null;
		case AtAround:
			springAdvice = new AspectJAroundAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
            // 不同切面通知对应不同Advice实现
		case AtBefore:
			springAdvice = new AspectJMethodBeforeAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
		case AtAfter:
			springAdvice = new AspectJAfterAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			break;
		case AtAfterReturning:
			springAdvice = new AspectJAfterReturningAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
			if (StringUtils.hasText(afterReturningAnnotation.returning())) {
				springAdvice.setReturningName(afterReturningAnnotation.returning());
			}
			break;
		case AtAfterThrowing:
			springAdvice = new AspectJAfterThrowingAdvice(
					candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
			AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
			if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
				springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
			}
			break;
		default:
			throw new UnsupportedOperationException(
					"Unsupported advice type on method: " + candidateAdviceMethod);
	}

	// Now to configure the advice...
	springAdvice.setAspectName(aspectName);
	springAdvice.setDeclarationOrder(declarationOrder);
	String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
	if (argNames != null) {
		springAdvice.setArgumentNamesFromStringArray(argNames);
	}
	springAdvice.calculateArgumentBindings();

	return springAdvice;
}
```

#### 挑选匹配的增强器

通过前面的逻辑，已经获取到切面类的可用增强器，接下来就是通过切点表达式来挑选出可以用在代理类上的增强器，代码入口 findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName)：

```java
protected List<Advisor> findAdvisorsThatCanApply(
		List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

	ProxyCreationContext.setCurrentProxiedBeanName(beanName);
	try {
		// 挑选出可以应用的增强器
		return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
	}
	finally {
		ProxyCreationContext.setCurrentProxiedBeanName(null);
	}
}
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
	if (candidateAdvisors.isEmpty()) {
		return candidateAdvisors;
	}
	List<Advisor> eligibleAdvisors = new ArrayList<>();
	for (Advisor candidate : candidateAdvisors) {
		// IntroductionAdvisor 类型的增强器，判断是否能应用于当前类
		if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
			eligibleAdvisors.add(candidate);
		}
	}
	boolean hasIntroductions = !eligibleAdvisors.isEmpty();
	for (Advisor candidate : candidateAdvisors) {
		if (candidate instanceof IntroductionAdvisor) {
			// already processed
			continue;
		}
		// 因为切面通知注解（Around,Before,After,AfterReturning,AfterThrowing）由
		// InstantiationModelAwarePointcutAdvisorImpl封装起来，该类又实现了PointcutAdvisor接口
		// 故五个切面注解都是通过canApply(candidate, clazz, hasIntroductions)来判断有哪些切面方法可以应用到当前类上
		if (canApply(candidate, clazz, hasIntroductions)) {
			eligibleAdvisors.add(candidate);
		}
	}
	return eligibleAdvisors;
}

```

至此，已经获取到适用的增强器，接下来就是创建代理的过程。

### 创建代理

代码入口 createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean))，实现逻辑如下：

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
		@Nullable Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

	// 创建代理工厂
	ProxyFactory proxyFactory = new ProxyFactory();
	// 获取当前类中的属性
	proxyFactory.copyFrom(this);

	// 如果代理工厂不属于代理目标类,将其转化成代理目标类（获取代理目标接口）
	if (!proxyFactory.isProxyTargetClass()) {
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			// 如果代理工厂类型不属于代理目标类,则添加代理目标接口
			// 获取代理接口
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}

	// 根据切面通知获取通知器（实际上,specificInterceptors持有的已经是Advisor类型 ）
	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	// 添加增强器
	proxyFactory.addAdvisors(advisors);
	// 设置要代理的目标类
	proxyFactory.setTargetSource(targetSource);
	// 空实现,留做扩展
	customizeProxyFactory(proxyFactory);
	// 控制代理工厂被配置后，是否还允许修改通知（默认不允许）
	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}
	// 获取代理
	return proxyFactory.getProxy(getProxyClassLoader());
}
```

主要做了以下内容：

- 创建代理工厂，获取当前类的属性；
- 添加代理接口，如果有的话；
- 添加切面增强器；
- 设置要代理的目标类；
- 获取代理。

#### 添加切面增强器

buildAdvisors(beanName, specificInterceptors)，实现逻辑：

```java
protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
	// Handle prototypes correctly...
	// 通用拦截器
	Advisor[] commonInterceptors = resolveInterceptorNames();

	List<Object> allInterceptors = new ArrayList<>();
	// 特定增强器（比如切面增强器）
	if (specificInterceptors != null) {
		allInterceptors.addAll(Arrays.asList(specificInterceptors));
		if (commonInterceptors.length > 0) {
			if (this.applyCommonInterceptorsFirst) {
				allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
			}
			else {
				allInterceptors.addAll(Arrays.asList(commonInterceptors));
			}
		}
	}
	if (logger.isTraceEnabled()) {
		int nrOfCommonInterceptors = commonInterceptors.length;
		int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
		logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
				" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
	}

	Advisor[] advisors = new Advisor[allInterceptors.size()];
	for (int i = 0; i < allInterceptors.size(); i++) {
        // 对增强器进行包装适配
		advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
	}
	return advisors;
}
public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
	// 如果本身是增强器(Advisor),直接返回
	if (adviceObject instanceof Advisor) {
		return (Advisor) adviceObject;
	}
	if (!(adviceObject instanceof Advice)) {
		throw new UnknownAdviceTypeException(adviceObject);
	}
	Advice advice = (Advice) adviceObject;
	if (advice instanceof MethodInterceptor) {
		// So well-known it doesn't even need an adapter.
        // 如果是方法拦截器,封装成默认增强器
		return new DefaultPointcutAdvisor(advice);
	}
	for (AdvisorAdapter adapter : this.adapters) {
		// Check that it is supported.
		if (adapter.supportsAdvice(advice)) {
			return new DefaultPointcutAdvisor(advice);
		}
	}
	throw new UnknownAdviceTypeException(advice);
}
```

获取代理，代码入口 proxyFactory.getProxy(getProxyClassLoader())，实现如下：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	return createAopProxy().getProxy(classLoader);
}
```

其实包含了两步：

- 创建 AOP 代理生成类（确定通过什么方式来生成代理）；
- 生成代理。

#### 获取代理生成类

查看 createAopProxy() 代码，如下：

```java
protected final synchronized AopProxy createAopProxy() {
	// 状态激活，表示创建一个新的 AOP 代理
	if (!this.active) {
		activate();
	}
	// 创建 AOP 代理
	return getAopProxyFactory().createAopProxy(this);
}
private void activate() {
	// 状态激活
	this.active = true;
	// 当代理创建的时候,监听器执行相应动作
	for (AdvisedSupportListener listener : this.listeners) {
		listener.activated(this);
	}
}
// 创建 AOP 代理
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	// 可能使用Cglib代理的选项：1.配置了Cglib代理优化;2.设置了proxyTargetClass=true;3.没有实现接口
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		// 目标类型是接口类型或本身是个代理类,依然使用 JDK 动态代理
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			return new JdkDynamicAopProxy(config);
		}
		// 没有实现接口使用 Cglib 代理
		return new ObjenesisCglibAopProxy(config);
	}
	else {
        // 其它情况直接使用 JDK 代理方式
		return new JdkDynamicAopProxy(config);
	}
}
```

#### 生成代理

AopProxy 接口的实现类分别是 ObjenesisCglibAopProxy 和 JdkDynamicAopProxy，由之前逻辑确定代理生成类后，这里就是直接调用 AopProxy.getProxy(classLoader) 来生成代理即可。

如果是 JDK 代理方式：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
	}
	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	// 调用 JDK API 获取动态代理
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

通过 JDK 方式生成的代理，方法的执行是在 invoke(proxy, method, args) 方法中，我们看下其实现：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;
	// 代理目标类
	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
        // 目标方法是 equals方法的处理
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
        // 目标方法是 hashCode方法的处理
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// There is only getDecoratedClass() declared -> dispatch to proxy config.
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// Get the interception chain for this method.
		// 获得当前方法的拦截器链
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			// 创建方法调用,拦截器的调用封装在 ReflectiveMethodInvocation 中
			MethodInvocation invocation =
					new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

我们看到，所有的拦截器都在这个方法里执行了，代码入口 this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)，实现如下：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
		Advised config, Method method, @Nullable Class<?> targetClass) {

	// This is somewhat tricky... We have to process introductions first,
	// but we need to preserve order in the ultimate list.
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	Advisor[] advisors = config.getAdvisors();
	List<Object> interceptorList = new ArrayList<>(advisors.length);
	Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	Boolean hasIntroductions = null;

	// 针对不同类型的增强器获取对应拦截器
	for (Advisor advisor : advisors) {
		if (advisor instanceof PointcutAdvisor) {
			// 切点增强器,看当前方法是否可用该增强器
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				boolean match;
				if (mm instanceof IntroductionAwareMethodMatcher) {
					if (hasIntroductions == null) {
						hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
					}
					match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
				}
				else {
					match = mm.matches(method, actualClass);
				}
				if (match) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					if (mm.isRuntime()) {
						// Creating a new object instance in the getInterceptors() method
						// isn't a problem as we normally cache created chains.
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}

	return interceptorList;
}
```

如果拦截器为空，直接执行方法调用，否则将代理类、目标类、方法及参数、拦截器等封装在 ReflectiveMethodInvocation类执行方法调用，具体实现在ReflectiveMethodInvocation.proceed()方法里，如下：

```java
public Object proceed() throws Throwable {
	// We start with an index of -1 and increment early.
	// 如果所有拦截器已处理完,最后执行切点方法
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    // 之前添加进来的增强器，最终转换对应拦截器，然后按顺序执行完毕
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		// Evaluate dynamic method matcher here: static part will already have
		// been evaluated and found to match.
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
		if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
			// 传递 this（当前对象） ，保证拦截器下标 currentInterceptorIndex 正确传递
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			// 动态匹配失败，跳过当前拦截器
			return proceed();
		}
	}
	else {
		// It's an interceptor, so we just invoke it: The pointcut will have
		// been evaluated statically before this object was constructed.
		// 传递 this（当前对象）,保证拦截器下标 currentInterceptorIndex 正确传递
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```

Cglib 方式生成代理，过程仅当了解，具体代码如下：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
	}

	try {
		Class<?> rootClass = this.advised.getTargetClass();
		Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

		Class<?> proxySuperClass = rootClass;
		if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
			proxySuperClass = rootClass.getSuperclass();
			Class<?>[] additionalInterfaces = rootClass.getInterfaces();
			for (Class<?> additionalInterface : additionalInterfaces) {
				this.advised.addInterface(additionalInterface);
			}
		}

		// Validate the class, writing log messages as necessary.
		validateClassIfNecessary(proxySuperClass, classLoader);

		// Configure CGLIB Enhancer...
		Enhancer enhancer = createEnhancer();
		if (classLoader != null) {
			enhancer.setClassLoader(classLoader);
			if (classLoader instanceof SmartClassLoader &&
					((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
				enhancer.setUseCache(false);
			}
		}
		enhancer.setSuperclass(proxySuperClass);
		enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

		Callback[] callbacks = getCallbacks(rootClass);
		Class<?>[] types = new Class<?>[callbacks.length];
		for (int x = 0; x < types.length; x++) {
			types[x] = callbacks[x].getClass();
		}
		// fixedInterceptorMap only populated at this point, after getCallbacks call above
		enhancer.setCallbackFilter(new ProxyCallbackFilter(
				this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
		enhancer.setCallbackTypes(types);

		// Generate the proxy class and create a proxy instance.
		return createProxyClassAndInstance(enhancer, callbacks);
	}
	catch (CodeGenerationException | IllegalArgumentException ex) {
		throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
				": Common causes of this problem include using a final class or a non-visible class",
				ex);
	}
	catch (Throwable ex) {
		// TargetSource.getTarget() failed
		throw new AopConfigException("Unexpected AOP exception", ex);
	}
}
```

## 总结

全部代码走完之后，梳理下动态 AOP 的实现：

- 如果配置文件中声明了 `<aop:aspectj-autoproxy>`,则用对应的标签解析器，注册 AOP 代理创建器的 beanDefinition（AnnotationAwareAspectJAutoProxyCreator），且该实现了 BeanPostProcessor  接口，故获取其它 bean 的时候会自动执行其代理逻辑；
- 如果需要创建代理类，则尝试获取增强器（包含切面类的各种切面通知，因为切面通知最后也是封装成了增强器），将增强器封装在代理生成类中；
- 按代理生成方式创建 JDK 动态代理或 Cglib 代理（默认是 JDK 动态代理）；
- 方法被调用的时候，通过代理来执行对应方法，执行方法前会调用拦截器链执行拦截器方法（拦截器链从增强器中提取出来），AOP 的相关切面方法也在这个过程中执行。