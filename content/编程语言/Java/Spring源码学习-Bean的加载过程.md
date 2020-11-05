---
title: "Spring源码学习-Bean的加载过程"
date: 2020-10-19T14:33:12+08:00
tags: ["Spring", "源码学习","IOC","bean的加载"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

## bean的加载

[源码Github地址](https://github.com/daigd/StudyDemo/tree/master/spring-source-learning)。

前面我们分析了`Spring`对XML配置信息的解析、注册过程，下面我们来看下`Bean`是如何加载的，首先来看下代码入口：

```java
BeanFactory bf = new XmlBeanFactory(new ClassPathResource("MyBeanFactoryTests.xml"));
MyTestBean bean = (MyTestBean) bf.getBean("myTestBean");
```

通过`XmlBeanFactory.getBean()`来获取，`Object getBean(String name) throws BeansException`是`BeanFactory` 接口定义的方法，具体实现在`AbstractBeanFactory`中，代码如下：

```java
public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		// 根据传入的 name 来提取真正的 beanName
		final String beanName = transformedBeanName(name);
		Object bean;

		// 尝试从缓存中加载单例
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
            // 对 bean 实例化
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
                // 如果存在依赖，则递归实例化依赖的 bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
                        // 缓存依赖调用 
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}
				// 实例化依赖的 bean后就可以实例化 mdb 本身了
				// Singleton 模式的创建
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// Prototype 模式的创建，每次都是 new 一个新实例
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
                    // 按指定scope模式创建bean
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 检查需要的类型是否符合创建 bean 的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

### bean加载的主要过程

由上述代码逻辑可知，`Spring`加载`Bean`的过程，大体上分为以下几个内容：

- 转换对应的beanName;
- 尝试从缓存中加载单例；
- 如果从单例缓存中获取到原始bean,对bean进行实例化；
- 原型模式创建阶段检查；
- 检测parentBeanFactory；
- 给当前bean指定创建标识；
- 将存储XML配置信息的 GernericBeanDefiniton 转换为 RootBeanDefinition;
- 寻找依赖，如果存在 bean 依赖，则实例化依赖 bean;
- 针对不同的 scope 进行 bean的创建；
- 类型检查及转换。

### FactoryBean 的使用

在分析`Bean`加载过程各个内容之前，我们先来了解下`Spring`中`FactoryBean`的作用。

定义如下：

```java
public interface FactoryBean<T> {
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

	@Nullable
	T getObject() throws Exception;
    
    @Nullable
	Class<?> getObjectType();

	default boolean isSingleton() {
		return true;
	}
}
```

关于它的作用，源码中描述为：

```java
If a bean implements this interface, it is used as a factory for an object to expose, not directly as a bean instance that will be exposed itself.
```

简单来说，如果一个`Bean`实现了`FactoryBean`接口，那么通过`getBean()`方法返回的不是`FactoryBean`本身，而是`FactoryBean.getObject()`方法所返回的对象，实际`Bean`的创建细节可以封装在`FactoryBean.getObject()`方法中，因此，通过这个工厂类接口，可以简化对复杂`Bean`的创建过程。

> 如果希望获取`FactoryBean`实例，那在使用`getBean(beanName)`方法时，在`beanName`前加上“&”前缀即可，例如`getBean("&person")`。

### 转换 BeanName

接下来我们来看转换`BeanName`的实现过程：

```java
// 根据传入的 name 来提取真正的 beanName
protected String transformedBeanName(String name) {
		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
	}
public String canonicalName(String name) {
		String canonicalName = name;
		// Handle aliasing...
    	// 为什么要做这个处理呢？因为 name 不一定是真正的 beanName,有可能是别名，所以循环去 aliasMap 按别名去取 beanName
    	// 直到取出来的 beanName 为null，说明此时传入 aliasMap 的 key就是真正的 beanName 了
		String resolvedName;
		do {
			resolvedName = this.aliasMap.get(canonicalName);
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}
```

### 缓存中获取单例 Bean

在`Spring`中，单例Bean只会创建一次，后续获取直接从缓存中获取，当然，这里先从缓存`singletonObjects`中加载，如果获取不到，再从`singletonFactories`中加载。

创建Bean的时候一般都会存在依赖注入，如果是单例Bean，为了避免循环依赖，`Spring`不等Bean创建完成就会将创建Bean的`ObjectFactory`提前加入到缓存中，一旦下一个Bean创建时需要依赖上一个Bean，则直接使用`ObjectFactory`；如果是原型Bean，则不允许循环依赖。

具体实现逻辑如下：

```java
public Object getSingleton(String beanName) {
    	// 参数 true 设置允许提前曝光引用
		return getSingleton(beanName, true);
}
// allowEarlyReference 默认允许提前曝光引用
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// singletonObjects = new ConcurrentHashMap<>(256);
	// 从单例缓存 singletonObjects 中，根据 beanName获取bean
		Object singletonObject = this.singletonObjects.get(beanName);
    // 如果为空，并且是创建单例对象，则锁定全局变量进行处理
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				//  earlySingletonObjects = new HashMap<>(16);
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
                    // 当某些 Bean 需要提前初始化的时候，会调用 addSingletonFactory 方法将对应的 ObjectFactory 初始化策略放在 singletonFactories 中
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
                        // 调用预先设定的getObject()方法
						singletonObject = singletonFactory.getObject();
                        // 记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
}
```

上面涉及到用于存储`Bean`的不同map，归纳如下：

- `singletonObject`：缓存 BeanName 和创建 Bean 实例之间的关系，bean name -> bean instance；
- `earlySingletonObjects`：作用同上，与`singletonObject`不同之处在于，当一个单例Bean被放在这里时，当Bean创建过程，就可以通过`getBean()`方法获取到了，目的是用来解决循环依赖；
- `singletonFactories`：缓存 BeanName 和创建 Bean 工厂之间的关系，同一个 BeanName ，不会同时在` earlySingletonObjects`和`singleFactories`存在映射。

`earlySingletonObjects`和`singletonFactories`主要用来解决`Spring`循环依赖的问题，具体可参考[Spring中循环引用的处理-1](https://www.iflym.com/index.php/code/201208280001.html)。

### 获取单例

上面的代码都是讲了从缓存中获取单例的过程，如果缓存中不存在已经加载的 Bean 实例，那就需要从头开始创建 Bean 实例了，`Spring`用`getSingleton`的重载方法来实现 Bean 的加载过程，如下：

```java
// 单例 Bean 的创建
if (mbd.isSingleton()) {
	sharedInstance = getSingleton(beanName, () -> {
		try {
            // 创建 Bean 的方法
			return createBean(beanName, mbd, args);
		}
		catch (BeansException ex) {
			// 创建出现异常，清除对应缓存，及其它清理工作
			destroySingleton(beanName);
			throw ex;
		}
	});
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}

protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	if (logger.isTraceEnabled()) {
		logger.trace("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// 获取 BeanClass,如果仍未解析BeanClass,在这里再次解析
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// 验证及准备覆盖的方法
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// 给BeanPostProcessors一个机会，用代理来替代真正的Bean实例
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	try {
        // 真正创建 Bean 的方法
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isTraceEnabled()) {
			logger.trace("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}
```

从代码中我们可以总结`createBean`完成的功能：

- 根据设置的 class 属性或根据 className 来解析 Class; 
- 对 override 属性进行标记和验证（针对配置的 lookup-method 和 replace-method 属性）；
- 应用 Bean 初始化前的后处理器，确认是否返回 Bean 实例代理；
- 创建 Bean。

#### 实例化前的前置处理

在真正调用`doCreateBean`方法之前，我们看到有调用这个`resolveBeforeInstantiation(beanName, mbdToUse)`方法，并且返回的`bean`如果不为空，则直接略过后续的 `Bean`创建过程，直接返回结果。我们看下其实现逻辑：

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
    // 如果尚未被解析
	if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
		// 确认 bean class此时已经解析完成
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            // 获取目标类型
			Class<?> targetType = determineTargetType(beanName, mbd);
			if (targetType != null) {
                // 实例化前的后处理器应用
                // 委托给 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation 来处理
				bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
				if (bean != null) {
                    // 实例化后的后处理器应用
                    // 委托给 BeanPostProcessor.postProcessAfterInitialization 来处理
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
		}
		mbd.beforeInstantiationResolved = (bean != null);
	}
	return bean;
}
```

#### 创建 Bean

当经历`resolveBeforeInstantiation`方法之后，如果返回的 Bean不为空，即创建了代理对象，则直接返回，否则需要进行常规 Bean 的创建。具体实现逻辑如下：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
    // 如果是单例，首先需要清除缓存
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
        // 根据指定 Bean 使用对应的策略创建新的实例：如工厂方法、构造函数自动注入、简单初始化等
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}

	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}

	// Eagerly cache singletons 可以解决循环依赖问题，这里判断是否需要提前曝光 单例缓存,条件是：单例&允许循环依赖&当前Bean正在创建中
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
        // 避免循环依赖，可以在 Bean 初始化完成前将创建实例的 ObjectFactory 加入缓存中，这里返回的Bean是初始状态的Bean
        // 如果实现了SmartInstantiationAwareBeanPostProcessor，则返回getEarlyBeanReference结果
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
        // 对 bean 进行填充，将各个属性注入，如果存在依赖于其它bean,则会递归初始化依赖的bean
		populateBean(beanName, mbd, instanceWrapper);
        // 调用初始化方法，比如 init-method
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
            // 如果 exposedObject 没有在初始化方法中被改变，即没有被增强
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
                    // 检测依赖
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
                // 因为 bean 创建后其所依赖的bean 一定是已经创建的，actualDependentBeans不为空，表明当前bean创建后其所依赖的bean没有全部创建完，也就是存在循环依赖，直接抛出异常
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
        // 注册 DisposableBean
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```

总结下上述代码流程：

- 如果是单例Bean，首先清除缓存；
- 实例化Bean，将BeanDefinition转化为BeanWrapper；
- MergedBeanDefinitionPostProcessor应用；
- 循环依赖处理；
- 将所有属性填充到Bean中；
- 注册 DisposableBean,以便于在销毁时调用；
- 完成创建并返回。

##### 创建 Bean 的实例

接下来我们深入分析 创建 `Bean`的每个步骤，针对单例`Bean`的清除缓存逻辑略过不提，我们先从实例化 `Bean`开始：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
    // 解析class
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

    // 如果用于创建Bean的回调方法不为空，直接用回调方法创建（Spring 5.0后才支持）
	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}

    // 如果工厂方法不为空，则使用工厂方法初始化策略
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
    // 一个类有多个构造函数，每个构造函数都有不同参数，所以调用前先根据参数锁定对应的构造函数或对应的工厂方法
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
    // 如果已经解析过则使用解析好的构造函数，不需要再次确定
	if (resolved) {
		if (autowireNecessary) {
            // 构造函数自动注入
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
            // 使用默认构造函数
			return instantiateBean(beanName, mbd);
		}
	}

	// 需要根据参数解析构造函数
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		// 构造函数自动注入
        return autowireConstructor(beanName, mbd, ctors, args);
	}

	// 获取指定的优先使用的构造函数
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}

	// 使用默认构造函数
	return instantiateBean(beanName, mbd);
}
```

上述代码主要做了以下工作：

- 解析BeanClass; 
- 如果有提供创建实例的Supplier接口函数，则使用该函数创建；
- 如果提供了工厂方法，则使用工厂方法创建实例；
- 解析构造函数并进行构造函数的实例化。

##### 带有参数的构造函数实例化

对于使用构造函数的实例化，有两种情况，一种是使用默认构造函数实例化，另一种是使用带有参数的构造器进行实例化。带有参数的实例化过程相当复杂，具体逻辑如下：

```java
protected BeanWrapper autowireConstructor(
		String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

	return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}

// ConstructorResolver.autowireConstructor 方法
public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
		@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

	BeanWrapperImpl bw = new BeanWrapperImpl();
	this.beanFactory.initBeanWrapper(bw);

	Constructor<?> constructorToUse = null;
	ArgumentsHolder argsHolderToUse = null;
	Object[] argsToUse = null;
	// explicitArgs 通过getBean方法传入，如果getBean方法调用的时候指定方法参数，则直接使用
	if (explicitArgs != null) {
		argsToUse = explicitArgs;
	}
	else {
        // 如果在getBean方法没有指定参数，则尝试从配置文件中获取
		Object[] argsToResolve = null;
        // 尝试从缓存中获取
		synchronized (mbd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse != null && mbd.constructorArgumentsResolved) {
				// Found a cached constructor...
                // 从缓存中获取
				argsToUse = mbd.resolvedConstructorArguments;
				if (argsToUse == null) {
                    // 配置的构造函数参数
					argsToResolve = mbd.preparedConstructorArguments;
				}
			}
		}
        // 如果缓存中存在
		if (argsToResolve != null) {
            // 解析参数类型，如 构造函数 A(int,int),通过此方法会把配置中的（“1”，“1”）转换成（1，1）
            // 缓存中的值可能是原始值，也可能是最终值
			argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
		}
	}
	// 没有被缓存的构造器和构造参数可使用
	if (constructorToUse == null || argsToUse == null) {
		// Take specified constructors, if any.
		Constructor<?>[] candidates = chosenCtors;
		if (candidates == null) {
			Class<?> beanClass = mbd.getBeanClass();
            // 如果没有给定构造方法，则从beanClass中获取
			try {
				candidates = (mbd.isNonPublicAccessAllowed() ?
						beanClass.getDeclaredConstructors() : beanClass.getConstructors());
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Resolution of declared constructors on bean Class [" + beanClass.getName() +
						"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
			}
		}
		// 只有一个构造方法且没有构造参数，使用无参构造器实例化，然后返回
		if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
			Constructor<?> uniqueCandidate = candidates[0];
			if (uniqueCandidate.getParameterCount() == 0) {
				synchronized (mbd.constructorArgumentLock) {
					mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
					mbd.constructorArgumentsResolved = true;
					mbd.resolvedConstructorArguments = EMPTY_ARGS;
				}
				bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
				return bw;
			}
		}
		// Need to resolve the constructor.
        // 有多个构造函数，需要处理
		boolean autowiring = (chosenCtors != null ||
				mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
		ConstructorArgumentValues resolvedValues = null;

		int minNrOfArgs;
		if (explicitArgs != null) {
			minNrOfArgs = explicitArgs.length;
		}
		else {
            // 从mdb中获取构造器参数
			ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            // 用于承载解析后的构造函数参数值
			resolvedValues = new ConstructorArgumentValues();
            // 解析到的参数个数
			minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
		}
		// 对给定的构造函数进行排序：public函数优先，形参数多的构造函数优先
		AutowireUtils.sortConstructors(candidates);
		int minTypeDiffWeight = Integer.MAX_VALUE;
		Set<Constructor<?>> ambiguousConstructors = null;
		LinkedList<UnsatisfiedDependencyException> causes = null;

		for (Constructor<?> candidate : candidates) {

			int parameterCount = candidate.getParameterCount();
			// 如果已经找到对应构造器，且参数个数大于构造器形参数，循环终止，因为构造器排序是按参数个数倒序排的
			if (constructorToUse != null && argsToUse != null && argsToUse.length > parameterCount) {
				// Already found greedy constructor that can be satisfied ->
				// do not look any further, there are only less greedy constructors left.
				break;
			}
            // 参数个数不相等，跳出当次循环
			if (parameterCount < minNrOfArgs) {
				continue;
			}

			ArgumentsHolder argsHolder;
			Class<?>[] paramTypes = candidate.getParameterTypes();
            // 获取到参数，则根据值构造对应参数类型的参数
			if (resolvedValues != null) {
				try {
                    // 通过ConstructorProperties注解获取参数名称
					String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, parameterCount);
					if (paramNames == null) {
                        // 获取参数名称解析器
						ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
						if (pnd != null) {
                            // 获取指定构造器的参数名称
							paramNames = pnd.getParameterNames(candidate);
						}
					}
                    // 根据参数名称、参数类型、参数值来创建参数持有者
					argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
							getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
				}
				catch (UnsatisfiedDependencyException ex) {
					if (logger.isTraceEnabled()) {
						logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
					}
					// Swallow and try next constructor.
					if (causes == null) {
						causes = new LinkedList<>();
					}
					causes.add(ex);
					continue;
				}
			}
			else {
				// Explicit arguments given -> arguments length must match exactly.
				if (parameterCount != explicitArgs.length) {
					continue;
				}
                // 构造函数没有参数的情况
				argsHolder = new ArgumentsHolder(explicitArgs);
			}
			// 检查是否有不确定性的构造函数存在
			int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
					argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
			// Choose this constructor if it represents the closest match.
            // 表示匹配最接近，选择作为构造函数
			if (typeDiffWeight < minTypeDiffWeight) {
				constructorToUse = candidate;
				argsHolderToUse = argsHolder;
				argsToUse = argsHolder.arguments;
				minTypeDiffWeight = typeDiffWeight;
				ambiguousConstructors = null;
			}
			else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
				if (ambiguousConstructors == null) {
					ambiguousConstructors = new LinkedHashSet<>();
					ambiguousConstructors.add(constructorToUse);
				}
				ambiguousConstructors.add(candidate);
			}
		}

		if (constructorToUse == null) {
			if (causes != null) {
				UnsatisfiedDependencyException ex = causes.removeLast();
				for (Exception cause : causes) {
					this.beanFactory.onSuppressedException(cause);
				}
				throw ex;
			}
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Could not resolve matching constructor " +
					"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
		}
		else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Ambiguous constructor matches found in bean '" + beanName + "' " +
					"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
					ambiguousConstructors);
		}

		if (explicitArgs == null && argsHolderToUse != null) {
            // 将解析的构造函数加入缓存中，封装在 mdb 类中
			argsHolderToUse.storeCache(mbd, constructorToUse);
		}
	}

	Assert.state(argsToUse != null, "Unresolved constructor arguments");
    // 将构建的实例加入 BeanWrapper 中
	bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
	return bw;
}
```

上面代码逻辑主要做了以下工作：

- 构造函数参数的确定：

  - 根据`explicitArgs`参数确定：如果`explicitArgs`参数不为空，那可以直接确定构造函数参数，因为它是在调用Bean的时候用户指定的，在`BeanFactory`中存在这样的方法，`explicitArgs`就是由`args`传进来的：

  ```java
  Object getBean(String name, Object... args) throws BeansException
  ```

  - 缓存中获取：如果之前已经解析过构造器和构造函数参数，那直接可以从缓存中获取；
  - 配置文件中获取：如果外部调用没有传入参数，且缓存中也没有获取成功，那只能从配置文件中分析得到了，已知`Sprin`中配置文件的数据最终都会被`BeanDefinition`实例承载，即`mbd`中包含，直接调用`mbd.preparedConstructorArguments`获取到原始参数，再通过`resolvePreparedArguments`解析获取到最终可用参数。

- 构造函数的确定：

  - 构造函数参数确定后，便可以根据参数在所有构造函数中来锁定对应的构造函数，通过`BeanClass`来获取所有构造函数，如果只有一个构造函数且参数为空，即可直接锁定使用的构造函数为无参构造函数；
  - 对构造函数进行排序， `public`函数优先，参数数目多的优先，然后遍历所有构造函数，根据实际参数的个数、参数名称、参数类型、参数值来匹配对应构造函数。

- 缓存构造函数和构造函数参数；

- 根据得到的构造函数及参数实例化 Bean。

##### 无参构造函数实例化

相比有参构造函数实例化，无参构造函数实例化的过程就比较简单了，具体代码如下：

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
	try {
		Object beanInstance;
		final BeanFactory parent = this;
        // 权限认证
		if (System.getSecurityManager() != null) {
			beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
					getInstantiationStrategy().instantiate(mbd, beanName, parent),
					getAccessControlContext());
		}
		else {
            // 直接调用实例化策略进行实例化
			beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
		}
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
	}
}
```

##### 实例化策略

`Bean`的实例化策略由方法`getInstantiationStrategy()`确定：

```java
protected InstantiationStrategy getInstantiationStrategy() {
		return this.instantiationStrategy;
}
// 在 AbstractAutowireCapableBeanFactory 中，instantiationStrategy 用 cglib 策略来初始化
private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();
```

`Spring`在得到确定的构造函数及参数后，对实例化的逻辑做了一次判断，如下：

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	// 如果没有重写方法，直接使用反射进行对象实例化
	if (!bd.hasMethodOverrides()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(
								(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
					}
					else {
						constructorToUse = clazz.getDeclaredConstructor();
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// 否则使用 cglib 动态代理来生成实例对象
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```

#### 属性注入

创建`Bean`实例完成后，接下来就是对属性进行注入，代码入口为`populateBean(beanName, mbd, instanceWrapper)`，具体实现如下：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	// 传入实例为空，不进行属性注入
    if (bw == null) {
		if (mbd.hasPropertyValues()) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
		}
		else {
			// Skip property population phase for null instance.
			return;
		}
	}

	// 给 InstantiationAwareBeanPostProcessors 最后一次机会修改 bean 的属性
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // bean 实例化后需要后置处理，则直接返回，不进行属性注入
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					return;
				}
			}
		}
	}

	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

	int resolvedAutowireMode = mbd.getResolvedAutowireMode();
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable.
        // 根据名称注入属性
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
        // 根据类型注入属性
		if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
	// 后置处理器已经初始化
	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    // 需要依赖检查
	boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

	PropertyDescriptor[] filteredPds = null;
	if (hasInstAwareBpps) {
		if (pvs == null) {
			pvs = mbd.getPropertyValues();
		}
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
				if (pvsToUse == null) {
					if (filteredPds == null) {
						filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
					}
                    // 对属性进行后置处理
					pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						return;
					}
				}
				pvs = pvsToUse;
			}
		}
	}
    // 依赖检查，对应depends-on 属性
	if (needsDepCheck) {
		if (filteredPds == null) {
			filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		}
		checkDependencies(beanName, mbd, filteredPds, pvs);
	}

	if (pvs != null) {
        // 将属性应用到 bean 实例中
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}
```

上面的代码主要做了以下几件事：

- 实例对象的非空校验，如果实例为空，不再进行后续处理；
- `InstantiationAwareBeanPostProcessor`对象控制程序是否继续进行属性填充；
- 根据注入类型选择按名称注入，或按类型注入；
- 对属性进行后置处理，及依赖检查；
- 将属性填充到 `BeanWrapper`中。

我们重点来分析下依赖注入及属性填充这两块逻辑。

##### 依赖按名称注入

代码入口`autowireByName(beanName, mbd, bw, newPvs)`，实现逻辑如下：

```java
protected void autowireByName(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	// 寻找 bw 中需要依赖注入的属性
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		if (containsBean(propertyName)) {
            // 递归初始化相关 bean
			Object bean = getBean(propertyName);
			pvs.add(propertyName, bean);
            // 注册依赖关系：将依赖关系缓存起来
			registerDependentBean(propertyName, beanName);
			if (logger.isTraceEnabled()) {
				logger.trace("Added autowiring by name from bean name '" + beanName +
						"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
						"' by name: no matching bean found");
			}
		}
	}
}
```

##### 依赖按类型注入

代码入口`autowireByType(beanName, mbd, bw, newPvs)`，实现逻辑如下：

```java
protected void autowireByType(
		String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}

	Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
    // 寻找 bw 中需要依赖注入的属性
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		try {
			PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
			// Don't try autowiring by type for type Object: never makes sense,
			// even if it technically is a unsatisfied, non-simple property.
			if (Object.class != pd.getPropertyType()) {
                // 获取指定属性的设值方法
				MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
				// Do not allow eager init for type matching in case of a prioritized post-processor.
				boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
				DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                
                // 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在 autowiredBeanNames 中，并属性存在多个封装 Bean 时，将其注入
				Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
				if (autowiredArgument != null) {
					pvs.add(propertyName, autowiredArgument);
				}
				for (String autowiredBeanName : autowiredBeanNames) {
                    // 注册依赖
					registerDependentBean(autowiredBeanName, beanName);
					if (logger.isTraceEnabled()) {
						logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
								propertyName + "' to bean named '" + autowiredBeanName + "'");
					}
				}
				autowiredBeanNames.clear();
			}
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
		}
	}
}
```

对于寻找类型匹配的逻辑封装在了`resolveDependency(desc, beanName, autowiredBeanNames, converter)`中，实现如下：

```java
// DefaultListableBeanFactory 类中
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
		@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

	descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
	if (Optional.class == descriptor.getDependencyType()) {
        // Optional 类注入的特殊处理
		return createOptionalDependency(descriptor, requestingBeanName);
	}
	else if (ObjectFactory.class == descriptor.getDependencyType() ||
			ObjectProvider.class == descriptor.getDependencyType()) {
        // ObjectFactory 类和 ObjectProvider 类注入的特殊处理
		return new DependencyObjectProvider(descriptor, requestingBeanName);
	}
	else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
        // javaxInjectProviderClass 类的特殊处理
		return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
	}
	else {
		Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
				descriptor, requestingBeanName);
		if (result == null) {
            // 通用处理逻辑
			result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
		}
		return result;
	}
}

// 通用处理逻辑
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
		@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

	InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
	try {
        // 如果能获取到对应Bean，直接返回
		Object shortcut = descriptor.resolveShortcut(this);
		if (shortcut != null) {
			return shortcut;
		}

		Class<?> type = descriptor.getDependencyType();
        // 对 @Value 注解的支持
		Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
		if (value != null) {
			if (value instanceof String) {
				String strVal = resolveEmbeddedValue((String) value);
				BeanDefinition bd = (beanName != null && containsBean(beanName) ?
						getMergedBeanDefinition(beanName) : null);
				value = evaluateBeanDefinitionString(strVal, bd);
			}
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			try {
				return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
			}
			catch (UnsupportedOperationException ex) {
				// A custom TypeConverter which does not support TypeDescriptor resolution...
				return (descriptor.getField() != null ?
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}
		}
		// 尝试获取注入的 Bean，这里注入类型有注入多个 bean 的情况
		Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
		if (multipleBeans != null) {
			return multipleBeans;
		}
		// 根据属性类型找到 beanFactory中所有类型匹配的bean
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        
		if (matchingBeans.isEmpty()) {
            // 如果 autowire 的require 属性为 true 而找到的匹配项为空，则抛出异常
			if (isRequired(descriptor)) {
				raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
			}
            // 否则对注入的 bean 返回 null
			return null;
		}

		String autowiredBeanName;
		Object instanceCandidate;

        // 处理匹配的 bean 有多个的情况
		if (matchingBeans.size() > 1) {
			autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
			if (autowiredBeanName == null) {
				if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
					return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
				}
				else {
					// In case of an optional Collection/Map, silently ignore a non-unique case:
					// possibly it was meant to be an empty collection of multiple regular beans
					// (before 4.3 in particular when we didn't even look for collection beans).
					return null;
				}
			}
			instanceCandidate = matchingBeans.get(autowiredBeanName);
		}
		else {
			// We have exactly one match.
            // 只有一个匹配项，直接使用
			Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
			autowiredBeanName = entry.getKey();
			instanceCandidate = entry.getValue();
		}

		if (autowiredBeanNames != null) {
			autowiredBeanNames.add(autowiredBeanName);
		}
		if (instanceCandidate instanceof Class) {
			instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
		}
        // 匹配的 bean 做下非空及类型校验
		Object result = instanceCandidate;
		if (result instanceof NullBean) {
			if (isRequired(descriptor)) {
				raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
			}
			result = null;
		}
		if (!ClassUtils.isAssignableValue(type, result)) {
			throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
		}
		return result;
	}
	finally {
		ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
	}
}

// DefaultListableBeanFactory.resolveMultipleBeans
// 处理多个 bean 的情况
private Object resolveMultipleBeans(DependencyDescriptor descriptor, @Nullable String beanName,
		@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {

    // 获取依赖的 class 类型
	final Class<?> type = descriptor.getDependencyType();

	if (descriptor instanceof StreamDependencyDescriptor) {
        // StreamDependencyDescriptor 寻找匹配的 bean
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
		if (autowiredBeanNames != null) {
			autowiredBeanNames.addAll(matchingBeans.keySet());
		}
		Stream<Object> stream = matchingBeans.keySet().stream()
				.map(name -> descriptor.resolveCandidate(name, type, this))
				.filter(bean -> !(bean instanceof NullBean));
		if (((StreamDependencyDescriptor) descriptor).isOrdered()) {
			stream = stream.sorted(adaptOrderComparator(matchingBeans));
		}
		return stream;
	}
    // 针对数组类型处理
	else if (type.isArray()) {
		Class<?> componentType = type.getComponentType();
		ResolvableType resolvableType = descriptor.getResolvableType();
		Class<?> resolvedArrayType = resolvableType.resolve(type);
		if (resolvedArrayType != type) {
			componentType = resolvableType.getComponentType().resolve();
		}
		if (componentType == null) {
			return null;
		}
        // 根据类型找到 beanFactory中所有匹配类型的 bean
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, componentType,
				new MultiElementDescriptor(descriptor));
		if (matchingBeans.isEmpty()) {
			return null;
		}
		if (autowiredBeanNames != null) {
			autowiredBeanNames.addAll(matchingBeans.keySet());
		}
		TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
		Object result = converter.convertIfNecessary(matchingBeans.values(), resolvedArrayType);
		if (result instanceof Object[]) {
			Comparator<Object> comparator = adaptDependencyComparator(matchingBeans);
			if (comparator != null) {
				Arrays.sort((Object[]) result, comparator);
			}
		}
		return result;
	}
    // 属性是 Collection 类型
	else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
		Class<?> elementType = descriptor.getResolvableType().asCollection().resolveGeneric();
		if (elementType == null) {
			return null;
		}
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, elementType,
				new MultiElementDescriptor(descriptor));
		if (matchingBeans.isEmpty()) {
			return null;
		}
		if (autowiredBeanNames != null) {
			autowiredBeanNames.addAll(matchingBeans.keySet());
		}
		TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
		Object result = converter.convertIfNecessary(matchingBeans.values(), type);
		if (result instanceof List) {
			if (((List<?>) result).size() > 1) {
				Comparator<Object> comparator = adaptDependencyComparator(matchingBeans);
				if (comparator != null) {
					((List<?>) result).sort(comparator);
				}
			}
		}
		return result;
	}
    // 属性是 map 类型
	else if (Map.class == type) {
		ResolvableType mapType = descriptor.getResolvableType().asMap();
		Class<?> keyType = mapType.resolveGeneric(0);
		if (String.class != keyType) {
			return null;
		}
		Class<?> valueType = mapType.resolveGeneric(1);
		if (valueType == null) {
			return null;
		}
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, valueType,
				new MultiElementDescriptor(descriptor));
		if (matchingBeans.isEmpty()) {
			return null;
		}
		if (autowiredBeanNames != null) {
			autowiredBeanNames.addAll(matchingBeans.keySet());
		}
		return matchingBeans;
	}
	else {
		return null;
	}
}
```

从上面的代码可以看到，寻找匹配类型的`Bean`的核心方法是`findAutowireCandidates(beanName, type, descriptor)`，我们看下它的具体实现：

```java
protected Map<String, Object> findAutowireCandidates(
		@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {
	// 找出同一类型的全部 beanName
	String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
			this, requiredType, true, descriptor.isEager());
	Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
    
	for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
		Class<?> autowiringType = classObjectEntry.getKey();
		if (autowiringType.isAssignableFrom(requiredType)) {
			Object autowiringValue = classObjectEntry.getValue();
            // 如果缓存的 map 中找对应类型的 注入 bean，则中断循环
			autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
			if (requiredType.isInstance(autowiringValue)) {
				result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
				break;
			}
		}
	}
    // 根据 beanName 获取 bean,通过beanFactory.getBean(beanName)方式获取
	for (String candidate : candidateNames) {
		if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
			addCandidateEntry(result, candidate, descriptor, requiredType);
		}
	}
	if (result.isEmpty()) {
		boolean multiple = indicatesMultipleBeans(requiredType);
		// Consider fallback matches if the first pass failed to find anything...
		DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
		for (String candidate : candidateNames) {
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor) &&
					(!multiple || getAutowireCandidateResolver().hasQualifier(descriptor))) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
		if (result.isEmpty() && !multiple) {
			// Consider self references as a final pass...
			// but in the case of a dependency collection, not the very same bean itself.
			for (String candidate : candidateNames) {
				if (isSelfReference(beanName, candidate) &&
						(!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
						isAutowireCandidate(candidate, fallbackDescriptor)) {
					addCandidateEntry(result, candidate, descriptor, requiredType);
				}
			}
		}
	}
	return result;
}
// 获取 bean 的逻辑封装在 addCandidateEntry(result, candidate, descriptor, requiredType) 中
private void addCandidateEntry(Map<String, Object> candidates, String candidateName,
		DependencyDescriptor descriptor, Class<?> requiredType) {

	if (descriptor instanceof MultiElementDescriptor) {
        // descriptor.resolveCandidate 实现就是通过 beanFactory.getBean(beanName) 获取对应 bean
		Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
		if (!(beanInstance instanceof NullBean)) {
			candidates.put(candidateName, beanInstance);
		}
	}
	else if (containsSingleton(candidateName) || (descriptor instanceof StreamDependencyDescriptor &&
			((StreamDependencyDescriptor) descriptor).isOrdered())) {
		Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
		candidates.put(candidateName, (beanInstance instanceof NullBean ? null : beanInstance));
	}
	else {
        // 最后是通过getType(beanName) 来获取 Bean
		candidates.put(candidateName, getType(candidateName));
	}
}
```

##### 属性填充到 `BeanWrapper`

程序获取对对象的属性后，属性是以`PropertyValues`形式存在的，还没有应用到实例化的 Bean 中，这一工作是在 `applyPropertyValues(beanName, mbd, bw, pvs)`完成的，看下其具体实现：

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	//属性值为空直接返回
    if (pvs.isEmpty()) {
		return;
	}

	if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
		((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
		if (mpvs.isConverted()) {
			// Shortcut: use the pre-converted values as-is.
            // 如果 mpvs 的值已经被转换为对应类型，那可以直接设置到 bw 中
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		original = mpvs.getPropertyValueList();
	}
	else {
        // pvs 不是MutablePropertyValues类型，直接使用原始的属性获取方法
		original = Arrays.asList(pvs.getPropertyValues());
	}

	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
    // 获取对应的解析器
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

	// Create a deep copy, resolving any references for values.
	List<PropertyValue> deepCopy = new ArrayList<>(original.size());
	boolean resolveNecessary = false;
    // 遍历属性，将属性转换成对应类型
	for (PropertyValue pv : original) {
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			Object originalValue = pv.getValue();
			if (originalValue == AutowiredPropertyMarker.INSTANCE) {
				Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
				if (writeMethod == null) {
					throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
				}
				originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
			}
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
			// Possibly store converted value in merged bean definition,
			// in order to avoid re-conversion for every created bean instance.
			if (resolvedValue == originalValue) {
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		mpvs.setConverted();
	}

	// Set our (possibly massaged) deep copy.
    // 最终存在 bw 的属性是以 MutablePropertyValues 包装起来的
	try {
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}
```

#### 对 Bean 应用自定义初始化方法

对`Bean`属性填充完成后，程序会调用用户设定的初始化方法，代码入口为`exposedObject = initializeBean(beanName, exposedObject, mbd)`,具体逻辑如下：

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			invokeAwareMethods(beanName, bean);
			return null;
		}, getAccessControlContext());
	}
	else {
        // 对特殊 bean 的处理：Aware，BeanClassLoaderAware，BeanFactoryAware
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
        //初始化前应用后置处理器
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
        // 激活用户自定义的 init 方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
        //初始化后应用后置处理器
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}

	return wrappedBean;
}
```

虽然该函数是对`Bean`应用自定义初始化方法，但是也做了其它工作：

- 激活Aware方法；
- 后置处理器的应用；
- 激活自定义的 init 方法。

##### 激活Aware方法

代码入口为`invokeAwareMethods(beanName, bean)`，实现如下：

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    // 如果bean实现了对应的 Aware 接口，分别对其进行赋值
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware) {
			ClassLoader bcl = getBeanClassLoader();
			if (bcl != null) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
			}
		}
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

##### 后置处理器的应用

调用自定义`init`方法前后都会应用后置处理器，这也是`Spring`可扩展性的体现，具体工作是委托给`BeanPostProcessor`接口来做，仅展示下`applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName)`的实现逻辑：

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
    // 调用 BeanPostProcessor 对应方法
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessBeforeInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

##### 激活自定义的init方法

自定义初始化方法除了使用配置`init-method`外，还可实现`InitializingBean`接口，`init-method`与`InitializingBean`接口的`afterPropertiesSet`方法都是在这里执行，如下代码所示：

```java
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
		throws Throwable {
	// 首先检查 bean 是否实现了 InitializingBean 接口，如果实现了，则执行其 afterPropertiesSet() 方法
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
		}
		if (System.getSecurityManager() != null) {
			try {
				AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
					((InitializingBean) bean).afterPropertiesSet();
					return null;
				}, getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
			((InitializingBean) bean).afterPropertiesSet();
		}
	}

	if (mbd != null && bean.getClass() != NullBean.class) {
		String initMethodName = mbd.getInitMethodName();
		if (StringUtils.hasLength(initMethodName) &&
				!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
				!mbd.isExternallyManagedInitMethod(initMethodName)) {
            // 调用用户自定义的初始化方法
			invokeCustomInitMethod(beanName, bean, mbd);
		}
	}
}
// AbstractAutowireCapableBeanFactory.invokeCustomInitMethod
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd)
		throws Throwable {
	// 获取初始化方法
	String initMethodName = mbd.getInitMethodName();
	Assert.state(initMethodName != null, "No init method set");
	Method initMethod = (mbd.isNonPublicAccessAllowed() ?
			BeanUtils.findMethod(bean.getClass(), initMethodName) :
			ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

	if (initMethod == null) {
		if (mbd.isEnforceInitMethod()) {
			throw new BeanDefinitionValidationException("Could not find an init method named '" +
					initMethodName + "' on bean with name '" + beanName + "'");
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No default init method named '" + initMethodName +
						"' found on bean with name '" + beanName + "'");
			}
			// Ignore non-existent default lifecycle methods.
			return;
		}
	}

	if (logger.isTraceEnabled()) {
		logger.trace("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
	}
	Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);

    // 通过反射调用，执行用户定义的方法
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			ReflectionUtils.makeAccessible(methodToInvoke);
			return null;
		});
		try {
			AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
					methodToInvoke.invoke(bean), getAccessControlContext());
		}
		catch (PrivilegedActionException pae) {
			InvocationTargetException ex = (InvocationTargetException) pae.getException();
			throw ex.getTargetException();
		}
	}
	else {
		try {
			ReflectionUtils.makeAccessible(methodToInvoke);
			methodToInvoke.invoke(bean);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}

```

#### 注册DisposableBean

现在创建`Bean`来到最后一步，如果需要销毁动作，注册`DisposableBean`，以在销毁`Bean`的时候执行对应逻辑，代码入口`registerDisposableBeanIfNecessary(beanName, bean, mbd)`，实现逻辑如下：

```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
	AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
	if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
		if (mbd.isSingleton()) {
			// Register a DisposableBean implementation that performs all destruction
			// work for the given bean: DestructionAwareBeanPostProcessors,
			// DisposableBean interface, custom destroy method.
            // 单例模式下注册需要销毁的Bean，此方法会处理实现 DisposableBean 接口的 bean，
            // 调用 DestructionAwareBeanPostProcessor 来处理
			registerDisposableBean(beanName,
					new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
		}
		else {
			// A bean with a custom scope...
            // 非单例情况的处理
			Scope scope = this.scopes.get(mbd.getScope());
			if (scope == null) {
				throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
			}
			scope.registerDestructionCallback(beanName,
					new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
		}
	}
}
```

### 注册单例

这一步完成之后，整个`Bean`的创建逻辑就完成了，接下来我们看下将单例`Bean`注册到缓存的过程，原代码如下：

```java
if (mbd.isSingleton()) {
    // getSingleton(String beanName, ObjectFactory<?> singletonFactory)
    // singletonFactory 的实现就是由 createBean(beanName, mbd, args) 完成
	sharedInstance = getSingleton(beanName, () -> {
		try {
			return createBean(beanName, mbd, args);
		}
		catch (BeansException ex) {
			// Explicitly remove instance from singleton cache: It might have been put there
			// eagerly by the creation process, to allow for circular reference resolution.
			// Also remove any beans that received a temporary reference to the bean.
			destroySingleton(beanName);
			throw ex;
		}
	});
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
// 获取单例 bean 及将单例注册到缓存
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
        // 确认下缓存中是否有该 bean
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
            // ObjectFactory.getObject() 获取创建完成的 bean,并标识其为新单例
			try {
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
                
                // 获取 bean 如果出现异常，再尝试从单例缓存中获取，如果为空直接报错
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
            // 将单例 bean 加入到 缓存中
			if (newSingleton) {
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
// 把 bean 加入到缓存中代码
// singletonObjects 保存全局注册的单例
// registeredSingletons 保存注册单例的beanName
// bean 创建完成后，只有 singletonObjects、registeredSingletons 保存单例信息，singletonFactories和earlySingletonObjects都会把单例记录移除
protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

### 从 Bean 的实例中获取对象

在 `getBean`方法中，`getObjectForBeanInstance`是个高频率使用的方法，我们得到`Bean`的实例后，首要就是调用这个方法来验证一下，主要是检测当前 Bean 类型是否是`FactoryBean`，如果是的话，调用`getObject`方法返回实例，否则直接返来原实例对象，具体实现如下：

```java
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

	// 如果 beanName 以 & 为前缀，表明想要获取 factoryBean 对象，则返回 该factoryBean
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		if (beanInstance instanceof NullBean) {
			return beanInstance;
		}
        // 如果 beanName 以 & 为前缀，获取的 Bean 类型不是 FactoryBean，则直接报错
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
		}
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		return beanInstance;
	}

	// 如果 bean 实例不是 FactoryBean 类型，也直接返回
	if (!(beanInstance instanceof FactoryBean)) {
		return beanInstance;
	}
	// 代码走到这里，说明当前 Bean 实例类型是 FactoryBean 了 
	Object object = null;
	if (mbd != null) {
		mbd.isFactoryBean = true;
	}
	else {
        // 如果 mbd 为空，尝试从缓存中加载 Bean
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
		// 将 Bean 实例转化为 FactoryBean
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// 如果 mbd 为空，从 BeanDefinition 缓存中获取
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
        // 确定 BeanDefinition 是用户定义还是应用声明的，如果是用户声明的，创建完对象后，不进行后处理操作（PostProcess）
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```

从上面代码中也可看出，`FactoryBean`生成对象的逻辑委托给了`getObjectFromFactoryBean`方法，总的来说，`getObjectForBeanInstance`主要做了下面几件事：

- 通过对`BeanName`的解析来判断是否返回`FactoryBean`实例；
- 对非`FactoryBean`对象不做任何处理，直接返回；
- 将`Bean`实例强转为`FactoryBean`对象，然后将从`FactoryBean`获取对象的工作委托给了`getObjectFromFactoryBean`。

我们接着来看`getObjectFromFactoryBean`的实现逻辑：

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
	if (factory.isSingleton() && containsSingleton(beanName)) {
        // 如果是单例 Bean，加锁保证全局唯一
		synchronized (getSingletonMutex()) {
            // 如果缓存中有记录，则获取到直接返回
			Object object = this.factoryBeanObjectCache.get(beanName);
			if (object == null) {
                // 真正获取对象的方法
				object = doGetObjectFromFactoryBean(factory, beanName);
				Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
				if (alreadyThere != null) {
					object = alreadyThere;
				}
				else {
					if (shouldPostProcess) {
						if (isSingletonCurrentlyInCreation(beanName)) {
							return object;
						}
                        // 将 beanName 添加到 singletonsCurrentlyInCreation 集合中，用于判断该Bean处于创建过程中
						beforeSingletonCreation(beanName);
						try {
                            // 如果单例Bean初始化完成，调用注册的 BeanPostProcessor 做后续处理工作
							object = postProcessObjectFromFactoryBean(object, beanName);
						}
						catch (Throwable ex) {
							throw new BeanCreationException(beanName,
									"Post-processing of FactoryBean's singleton object failed", ex);
						}
						finally {
                            // 将 BeanName 从 singletonsCurrentlyInCreation 集合中移除，标识 Bean 创建完成
							afterSingletonCreation(beanName);
						}
					}
                    // 将单例 Bean 缓存到 factoryBeanObjectCache中
					if (containsSingleton(beanName)) {
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
			}
			return object;
		}
	}
	else {
        // 非单例 Bean 的生成，每次都是重新创建，不需要缓存
		Object object = doGetObjectFromFactoryBean(factory, beanName);
		if (shouldPostProcess) {
			try {
				object = postProcessObjectFromFactoryBean(object, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
			}
		}
		return object;
	}
}
```

上面代码中其实还没进入到生成对象的逻辑里面，如果`shouldPostProcess`参数为真，就对生成对象做一些增强操作，除此之外，还对单例Bean做了全局唯一保证及缓存处理，所以，继续跟踪真正的处理逻辑：

```java
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
		throws BeanCreationException {

	Object object;
	try {
        // 是否需要权限校验
		if (System.getSecurityManager() != null) {
			AccessControlContext acc = getAccessControlContext();
			try {
				object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
            // 直接调用 factory.getObject() 获取实际想要的 Bean 对象
			object = factory.getObject();
		}
	}
	catch (FactoryBeanNotInitializedException ex) {
		throw new BeanCurrentlyInCreationException(beanName, ex.toString());
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
	}

	// 如果对象在创建过程中，getObject()获取到空，直接报错
	if (object == null) {
		if (isSingletonCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(
					beanName, "FactoryBean which is currently in creation returned null from getObject");
		}
		object = new NullBean();
	}
	return object;
}
```

### 获取原型对象

分析完单例的获取、注册过程，对比原型对象的获取代码不难发现，对于原型对象来说，`Bean`创建完成之后不进行缓存登记逻辑，即每次都是新对象，代码分析过程就不展开了。

### 返回Bean类型校验

`Bean`加载完毕返回之前，还有一步工作要做，如果传入的类型参数不为空，还需要检查创建的`Bean`是否符合指定，如果不符合，直接抛出异常，否则，直接返回当前`Bean`。



