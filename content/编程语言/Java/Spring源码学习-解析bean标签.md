---
title: "Spring源码学习-解析bean标签"
date: 2020-10-16T17:23:12+08:00
tags: ["Spring", "源码学习","IOC","解析bean标签"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

## 默认标签处理

Spring 对 XML 默认标签的处理逻辑在`DefaultBeanDefinitionDocumentReader` 类中，代码逻辑如下：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	// 对 beans 处理，默认标签解析，通过对命名空间的判断来区分	
    if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
                    // root 元素下可能有自定义标签，所以需要再做一次判断
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
            // 自定义标签解析
			delegate.parseCustomElement(root);
		}
}
```

继续从 `parseDefaultElement(ele, delegate)` 点进去，

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	// 解析 import 标签	
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
    // 解析 alias 标签
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
    // 解析 bean 标签
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
    // 解析 beans 标签
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

### bean 标签解析

由于 `bean` 标签解析的过程比较复杂，着重来分析这个过程，

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	// 委托 BeanDefinitionParserDelegate 进行元素解析
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
            // 若标签的子节点下还有自定义属性，还需要再次对自定义标签进行解析
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 对Bean进行注册
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 发送响应事件，通知相关监听器，Bean已经注册完成
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

下面我们针对各个操作来进行分析，首先从元素解析和提取信息开始,进入 `delegate.parseBeanDefinitionElement(ele)` 实现：

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
}

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
	// 解析 id 属性	
    String id = ele.getAttribute(ID_ATTRIBUTE);
    // 解析 name 属性
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	List<String> aliases = new ArrayList<>();
    // 用 ,; 分割 name,当成别名
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

    // 不指定 id　属性，则以第一个别名当成bean id
	String beanName = id;
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		beanName = aliases.remove(0);
		if (logger.isTraceEnabled()) {
			logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
		}
	}

	if (containingBean == null) {
        // 验证 name 是否使用过
		checkNameUniqueness(beanName, aliases, ele);
	}
	// 除了 beanName和alias，对其它元素进行解析，后续分析见下文，解析过程出错，会返回null
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	// 如果解析成功，则 beanDefinition 不为 null
    if (beanDefinition != null) {
        // 如果没有指定 beanName，按指定规则生成
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
				}
					
			}
			catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
        // 将获取到数据封装在 BeanDefinitionHolder 中
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}

	return null;
}
```

下面继续跟踪元素解析过程，从 `parseBeanDefinitionElement(ele, beanName, containingBean)` 点进去：

```java
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {
		// ParseState 标识元素解析状态
		this.parseState.push(new BeanEntry(beanName));

		String className = null;
    	// 解析 class 属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
    	// 解析 parent 属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
            // 创建保存 Bean 属性的 GenericBeanDefinition， beanClass保存根据className 创建的实例，如果创建失败，则保存 className
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			// 这里对其它 Bean 属性进行解析
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            // 解析 description 属性
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
			// 解析 meta 元数据
			parseMetaElements(ele, bd);
            // 解析 lookup-method 属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            // 解析 replaced-method 属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

            // 解析构造器参数
			parseConstructorArgElements(ele, bd);
            // 解析 property 子元素
			parsePropertyElements(ele, bd);
            // 解析 qualifier 子元素
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

通过上述代码逻辑可以得知，`Spring` 通过 `BeanDefinition` 将配置中的 Bean 信息转换为容器内部表示，创建的实例对象类型为`GenericBeanDefinition`，我们来看下`AbstractBeanDefinition bd = createBeanDefinition(className, parent)` 逻辑的具体实现：

```java
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {

		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}

public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
    	// parentName 可能为空
		bd.setParentName(parentName);
		if (className != null) {
            // 如果 classLoader 不为空，则使用传入的 classLoader 同一虚拟机加载类对象，否则只是记录 className
			if (classLoader != null) {
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			}
			else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}
```

而`parseBeanDefinitionAttributes(ele, beanName, containingBean, bd)`是对Bean其它元素的解析，代码不复杂，具体如下：

```java
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
		// singleton 属性废弃使用
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
    	// 解析 scope 属性
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
    	// 如果未指定 scope属性，且嵌入了容器Bean,则以嵌入的Bean scope为准
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
			bd.setScope(containingBean.getScope());
		}

		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
    	// 如果没有设置 懒加载或设置成其它字符，该属性都被设置成false
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

    	// 剩余属性解析过程也不复杂，就不赘述了
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));

		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if (isDefaultValue(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}

		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}

		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}

		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```

经过这些步骤之后，XML 上的 Bean 配置都已经保存在了 `AbstractBeanDefinition`中，我们回过头来看下代码入口：

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    	// 完成了 Bean 配置的解析
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
            // 如果 Bean 使用的是默认的标签配置，但是其中的子元素却使用了自定义配置时，这行代码就会起作用了，要注意，这里的自定义类型并不是针对
            // Bean的，而是针对属性，下面我们来看下它的具体实现
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}

```

`delegate.decorateBeanDefinitionIfRequired(ele, bdHolder)`具体实现：

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
		return decorateBeanDefinitionIfRequired(ele, originalDef, null);
	}

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = originalDef;

		// Decorate based on custom attributes first.
    	// 遍历所有属性，看是否有适用于修饰的属性
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
    	// 遍历所有子节点，看是否有适用于修饰的子元素
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
```

对`decorateIfRequired(node, finalDefinition, containingBd)`的实现看一下，

```java
public BeanDefinitionHolder decorateIfRequired(
			Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		String namespaceUri = getNamespaceURI(node);
    	// 通过命名空间来判断，如果是自定义标签才进行解析
		if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
            // 根据命名空间找到对应的处理器
			NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
			if (handler != null) {
                // 进行修饰，具体解析逻辑委托给对应处理器
				BeanDefinitionHolder decorated =
						handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
				if (decorated != null) {
					return decorated;
				}
			}
			else if (namespaceUri.startsWith("http://www.springframework.org/schema/")) {
				error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
			}
			else {
				// A custom namespace, not to be handled by Spring - maybe "xml:...".
				if (logger.isDebugEnabled()) {
					logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
				}
			}
		}
		return originalDef;
	}
```

至此，对于配置文件，解析默认标签完成，对于自定义标签的修饰，通过命名空间来找到对应处理器来进行修饰，对于得到的 `BeanDefinition`已经可以满足后续使用要求，接下来就是对 `Bean` 进行注册，我们看下对应代码`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())`的实现：

```java
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
    	// 使用 BeanName 做唯一标识注册
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
    	// 注册全部别名
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

具体注册逻辑，我们通过`registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())` 来看一下，参数`BeanDefinitionRegistry`是一个接口，具体实现由`getReaderContext().getRegistry()`指定，`getReaderContext()`返回的是`XmlReaderContext`，它对资源的处理、监听器、资源加载、命名空间处理器等进行了封装，`getRegistry`返回的则是`XmlBeanDefinitionReader`的注册器，初始化`XmlBeanFactory`的时候就同时对`XmlBeanDefinitionReader`进行了加载：

```java
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
```

因为返回的`getRegistry()`其实是由`XmlBeanFactory`继承过来的`DefaultListableBeanFactory`，对应的`registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition())`实现如下：

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {
		// 参数校验
		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    // AbstractBeanDefinition 的实现类有：AnnotatedGenericBeanDefinition，ChildBeanDefinition，RootBeanDefinition，GenericBeanDefinition，通过之前代码分析，读取XML的配置信息最后是封装成了GenericBeanDefinition
		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
                // 校验methodOverrides是否与工厂方法并存，并存则报错；或methodOverrides对应的方法是否存在，不存在则报错
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

    // beanDefinitionMap 的定义：private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    // 根据 beanName去查询 Bean 是否存在
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		if (existingDefinition != null) {
            // 如果对应 Bean 已存在，判断是否允许覆盖，不允许则抛出异常，默认允许
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
            // 将当前 Bean 加入 map 缓存
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
            // 检查 该Bean的工厂Bean 创建阶段是否已经开始，如果已经开始 
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
                    // Bean 加入 map 缓存
					this.beanDefinitionMap.put(beanName, beanDefinition);
                    // 更新注册的beanDefinitionNames集合
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
                    // 清除之前的BeanName缓存
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
            // 重设所有 beanName 对应的缓存
			resetBeanDefinition(beanName);
		}
    	// 如果为Bean 类型配置元数据缓存，也进行清理
		else if (isConfigurationFrozen()) {
			clearByTypeCache();
		}
	}
```

看完上面的代码，对`BeanDefinition` 的注册其实就是放入到`map` 缓存中，同时做了一些额外操作，比如对`methodsOverrides`属性的校验，`Bean`是否允许覆盖的校验，beanName 缓存的更新等。

接下来，我们看下别名的注册过程：

```java
String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}

public void registerAlias(String name, String alias) {
    	// beanName和别名进行非空判断
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
    // aliasMap的定义：private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);
		synchronized (this.aliasMap) {
			if (alias.equals(name)) {
                // 别名如果和beanName一样，则移除该别名
				this.aliasMap.remove(alias);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
				}
			}
			else {
				String registeredName = this.aliasMap.get(alias);
				if (registeredName != null) {
                    // 如果别名对应的BeanName和当前传进来的name一样，则不需要注册
					if (registeredName.equals(name)) {
						// An existing alias - no need to re-register
						return;
					}
					if (!allowAliasOverriding()) {
						throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
								name + "': It is already registered for name '" + registeredName + "'.");
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
								registeredName + "' with new target name '" + name + "'");
					}
				}
                //别名循环检查： 如果有A对应B: A->B,同时有A对应C,C又对应B的映射情况： A->C->B,则会抛出异常
				checkForAliasCircle(name, alias);
                // 注册别名:key为别名，value为BeanName
				this.aliasMap.put(alias, name);
				if (logger.isTraceEnabled()) {
					logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
				}
			}
		}
	}
```

至此，`Bean`和别名的注册过程已经梳理完毕，之后就是通知监听器解析和注册动作已完成，由对应监听器做一些后续处理工作，`getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder))`,因为`ReaderEventListener`的默认实现为`EmptyReaderEventListener`，即`Spring`对此事件没有做任何处理，所以这里的实现只为扩展。

### import、alias、beans标签的解析

import、alias、beans标签的解析过程相比bean标签的解析要简单，此处不再展开。



