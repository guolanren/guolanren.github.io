---
title: Spring-BeanDefinition的加载
description: 通常使用 Spring 开发，需要指定 spring 的配置文件，里面定义了开发中有可能需要用到的 Bean。Spring 通过 BeanDefinitionReader，读取配置文件，将 Bean(定义)加载到容器中。BeanDefinitionReader 支持 *.xml、*.properties、*.groovy 三种配置文件的读取。其中 xml 配置文件方式较为常见，下文将以 XmlBeanDefinitionReader 介绍 BeanDefinitionReader。
categories: 
 - code
tags:
 - spring
 - 源码
 - spring源码
---

------

## 前言

​	通常使用 **Spring** 开发，需要指定 **spring** 的配置文件，里面定义了开发中有可能需要用到的 **Bean**。**Spring** 通过 `BeanDefinitionReader`，读取配置文件，将 **Bean(定义)** 加载到容器中。`BeanDefinitionReader` 支持 `*.xml`、`*.properties`、`*.groovy` 三种配置文件的读取。其中 **xml** 配置文件方式较为常见，下文将以 `XmlBeanDefinitionReader` 介绍 `BeanDefinitionReader`。

## 类图

![BeanDefinitionReader](https://github.com/guolanren/gallery/blob/master/found/2020-05-15-Spring-Bean%E5%AE%9A%E4%B9%89%E7%9A%84%E5%8A%A0%E8%BD%BD/BeanDefinitionReader.png?raw=true)

## Start

```java
public class BeanFactoryStart {
    
    public static void main(String[] args) {
        BeanFactory bf = new DefaultListableBeanFactory();
        BeanDefinitionReader bdr = new XmlBeanDefinitionReader((BeanDefinitionRegistry) bf);
        // 1. 使用 AbstractBeanDefinitionReader 的 loadBeanDefinitions(String location)
        bdr.loadBeanDefinitions("classpath:bean-factory.xml");
        // 2. 使用 XmlBeanDefinitionReader 重写后的 loadBeanDefinitions(Resource resource)
        Resource resource = new ClassPathResource("bean-factory.xml");
        bdr.loadBeanDefinitions(resource);
    }
}
```

## 构建 XmlBeanDefinitionReader

```java
/**
 * XmlBeanDefinitionReader 唯一的构造器。
 * 需要指定 BeanDefinitionRegistry，读取到的 Bean 定义，将转为 BeanDeinitionRegistry 对象，进行注册。
 * 实际调用父类 AbstractBeanDefinitionReader 的构造器完成初始化。
 */
public XmlBeanDefinitionReader(BeanDefinitionRegistry registry) {
    super(registry);
}

protected AbstractBeanDefinitionReader(BeanDefinitionRegistry registry) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;

    // Determine ResourceLoader to use.
    // 入参 registry 是一个 ResourceLoader 的话，则使用它作为 ResourceLoader(资源加载器)。
    // 一般指的是 ApplicationContext，它相对于 BeanFactory 拓展了 ResourceLoader 等一些功能。
    if (this.registry instanceof ResourceLoader) {
        this.resourceLoader = (ResourceLoader) this.registry;
    }
    // 入参 registry 只是一个普通的 BeanDefinitionRegistry，则使用默认的 PathMatchingResourcePatternResolver 作为 ResourceLoader(资源加载器)。
    else {
        this.resourceLoader = new PathMatchingResourcePatternResolver();
    }

    // Inherit Environment if possible
    // 同样的，入参 registry 是一个 EnvironmentCapable 的话，则使用它的 Environment。
    if (this.registry instanceof EnvironmentCapable) {
        this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
    }
    // 否侧，创建一个默认的 StandardEnvironment。
    else {
        this.environment = new StandardEnvironment();
    }
}
```

## 加载 BeanDefinition

​	**Start** 中使用了两种方式加载：**1.** 直接指定配置文件路径。 **2.** 将配置文件封装成 `Resource`，再作为入参加载。

### 概念

​	在介绍如何加载 `BeanDefinition` 之前，有必要先了解一下 `Resource`、`Document`。其实配置文件中的 **Bean** 定义不是直接就被读取、解析成 `BeanDefintion` 对象的。`XmlBeanDefinitionReader` 的大致工序是：配置文件(`bean-definition-reader.xml`) -> `Resource` -> `InputSource` -> `Document` -> `BeanDefinition`。

- `Resource`

​	`Resource` 是 **Spring** 内部的资源抽象，封装了底层资源。**Spring** 提供了多种资源不同来源的实现，**File**、**Classpath**、**URL**、**ByteArray**...

![Resource](https://github.com/guolanren/gallery/blob/master/found/2020-05-15-Spring-Bean%E5%AE%9A%E4%B9%89%E7%9A%84%E5%8A%A0%E8%BD%BD/Resource.png?raw=true)

- `InputSource`

  `org.xml.sax` 包下的接口，用于定义一个 **XML** 实体。`javax.xml.parsers.DocumentBuilder` 通过解析 `InputSource` 构建 `Document`。

- `Document`

  `org.w3c.dom` 包下的接口。**DOM** 是 **Document Object Model**（文档对象模型）的缩写，定义了如何访问 **HTML** 、**XML** 文档的标准。

### 解析 location 成 Resource

```java
/**
 * org.springframework.beans.factory.support.AbstractBeanDefinitionReader
 */
@Override
public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(location, null);
}

/**
 * org.springframework.beans.factory.support.AbstractBeanDefinitionReader
 */
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    	// 获取 ResourceLoader。
    	// 构造 BeanDefinitionReader 时进行过设置，
    	// 使用 registry 作为 ResourceLoader 或者创建一个 PathMatchingResourcePatternResolver，
    	// 如果 registry 不是 ResourceLoader。
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

    	// PathMatchingResourcePatternResolver 就是一个 ResourcePatternResolver。
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
                // 可以通过模式去匹配多个配置文件，封装成 Resource。
                Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
                // 使用重载方法，传入 Resources 加载 BeanDefinition。
                int loadCount = loadBeanDefinitions(resources);
                // actualResources 是一个存储了已经被处理过的 Resource 集合，可以用来指示调用者忽略那些 Resource。
                if (actualResources != null) {
                    for (Resource resource : resources) {
                        actualResources.add(resource);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
                }
                // 返回加载 BeanDefinition 的数量。
                return loadCount;
            }
            catch (IOException ex) {
                throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
            }
        }
    	// 如果 resourceLoader 不是 ResourcePatternResolver。
    	else {
            // Can only load single resources by absolute URL.
            // 这个解析器只能加载一个绝对路径 URL 资源。
            Resource resource = resourceLoader.getResource(location);
            int loadCount = loadBeanDefinitions(resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
            }
            return loadCount;
        }
}
```

### Resource 转 InputSource

```java
/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    // 使用默认编码，字符集封装 Resource。
    return loadBeanDefinitions(new EncodedResource(resource));
}

/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " + encodedResource);
    }

    // 获取当前线程中，正在被加载的 Resource 集合(ThreadLoad 变量)
    // 用于检测配置文件是否存在循环 import。
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    // 如果当前线程还没有设置 currentResources，则初始化一个。
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    // 添加失败意味着已经循环 import
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            // 构建 InputSource，后面 DocumentBuilder 解析的 xml 文件就是被封装成 InputSource。
            InputSource inputSource = new InputSource(inputStream);
            // 设置编码，有的话。
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 执行真正的加载 BeanDefinition 方法。
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        // 加载完毕，从正在被加载集合中移除。
        currentResources.remove(encodedResource);
        // 如果 currentResources 已经为空了，使用 ThreadLocal#remove() 移除，避免内存泄漏。
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}
```

### 解析 InputSource 成 Document

```java
/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
        // 解析 InputSource，加载 Document。
        Document doc = doLoadDocument(inputSource, resource);
        return registerBeanDefinitions(doc, resource);
    }
    catch (BeanDefinitionStoreException ex) {
        throw ex;
    }
	...
}

/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                                            getValidationModeForResource(resource), isNamespaceAware());
}

/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
protected int getValidationModeForResource(Resource resource) {
    // 如果手动设置了 xml 文件的验证模式，则使用手动的验证模式。
    int validationModeToUse = getValidationMode();
    if (validationModeToUse != VALIDATION_AUTO) {
        return validationModeToUse;
    }
    // 解析配置文件，获取验证模式。
    int detectedMode = detectValidationMode(resource);
    if (detectedMode != VALIDATION_AUTO) {
        return detectedMode;
    }
    // Hmm, we didn't get a clear indication... Let's assume XSD,
    // since apparently no DTD declaration has been found up until
    // detection stopped (before finding the document's root tag).
    // 默认使用 XSD 验证模式。
    return VALIDATION_XSD;
}

/**
 * org.springframework.beans.factory.xml.DefaultDocumentLoader
 */
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
                             ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
    // 创建 DocumentBuilder 工厂。
    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    if (logger.isDebugEnabled()) {
        logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
    }
    // 构建 DocumentBuilder。
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    // 解析 InputSource 成 Document。
    return builder.parse(inputSource);
}
```

### 解析 Document

```java
/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
    throws BeanDefinitionStoreException {
    try {
        Document doc = doLoadDocument(inputSource, resource);
        // 使用 Document 注册 BeanDefinition。
        return registerBeanDefinitions(doc, resource);
    }
    catch (BeanDefinitionStoreException ex) {
        throw ex;
    }
	...
}

/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 创建一个 BeanDefinitionDocumentReader 用于读取 Document。
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    // 先获取注册表中已有的 BeanDefinition 数目。
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 注册 BeanDefinition。
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    // 计算出本次添加的 BeanDefinition 数目。
    return getRegistry().getBeanDefinitionCount() - countBefore;
}

/**
 * org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 */
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    // 获取配置文件根节点。
    Element root = doc.getDocumentElement();
    // 使用根节点进行 BeanDefinition 注册。
    doRegisterBeanDefinitions(root);
}

/**
 * org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
 */
protected void doRegisterBeanDefinitions(Element root) {
    // Any nested <beans> elements will cause recursion in this method. In
    // order to propagate and preserve <beans> default-* attributes correctly,
    // keep track of the current (parent) delegate, which may be null. Create
    // the new (child) delegate with a reference to the parent for fallback purposes,
    // then ultimately reset this.delegate back to its original (parent) reference.
    // this behavior emulates a stack of delegates without actually necessitating one.
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

    // 是默认的命名空间吗：http://www.springframework.org/schema/beans。
    if (this.delegate.isDefaultNamespace(root)) {
        // 获取 profile 属性。
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            // profile 属性转数组。允许[,; ](逗号，分号，空格)分隔。
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            // 如果 Environment 不支持 profile，直接跳过不加载。
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                }
                return;
            }
        }
    }

    // 前置处理，空方法，可以继承 DefaultBeanDefinitionDocumentReader 重写。
    preProcessXml(root);
    // 解析 BeanDefinition。
    parseBeanDefinitions(root, this.delegate);
    // 后置处理，空方法，可以继承 DefaultBeanDefinitionDocumentReader 重写。
    postProcessXml(root);

    this.delegate = parent;
}

/**
 * org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
 * 解析 <import>、<alias>、<bean> 标签
 */
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // 是默认的命名空间吗：http://www.springframework.org/schema/beans。
    if (delegate.isDefaultNamespace(root)) {
        // 获取 Node 列表。
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            // 遍历 <beans> 下的子标签，<import>、<alias>、<bean>...
            Node node = nl.item(i);
            // 如果是一个标签节点。
            if (node instanceof Element) {
                Element ele = (Element) node;
                // 默认标签解析。
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                // 自定义标签解析。
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    // 自定义标签解析。
    else {
        delegate.parseCustomElement(root);
    }
}

/**
 * org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
 */
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    // <import> 标签。
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        // 使用对应的配置文件，加载 BeanDefinition。
        // 如果存在配置文件循环 import, 前面 Resource 转 InputSource 会检测出来抛出异常。
        importBeanDefinitionResource(ele);
    }
    // <alias> 标签解析。
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    // <bean> 标签解析。
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        // 处理 BeanDefinition 注册。
        processBeanDefinition(ele, delegate);
    }
    // 内嵌 <beans> 标签解析。
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        // 递归注册 BeanDefinition。
        doRegisterBeanDefinitions(ele);
    }
}

/**
 * org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
 */
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

/**
 * org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
 */
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    }
    // 自定义标签的命名空间处理器。
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    // 特定处理器解析特定标签。
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

### 注册 BeanDefinition

```java
/**
 * org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
 */
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 根据 Element 构建出 BeanDefinitionHolder。
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            // 使用工具类 BeanDefinitionReaderUtils 注册 BeanDefinition。
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                                     bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        // 发送注册事件，通知相关的监听器，这个 Bean 已经加载好了。
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}

/**
 * org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
 */
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}

/**
 * org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
 */
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    // 获取 <bean> 的 id。
    String id = ele.getAttribute(ID_ATTRIBUTE);
    // 获取 <bean> 的 name。
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        // 将 name 按合法分隔符分隔，作为 bean 的别名。
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    // id 才是 bean 的名字。
    String beanName = id;
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        // 如果没有指定 id，将使用 name 中的第一个名字作为 id，并将收个 name 剔除别名列表。
        beanName = aliases.remove(0);
        if (logger.isDebugEnabled()) {
            logger.debug("No XML 'id' specified - using '" + beanName +
                         "' as bean name and " + aliases + " as aliases");
        }
    }

    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // 进一步解析 BeanDefinition。
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        // 如果 beanName 还是为空。
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }
                // 如果 id 和 name 都没有指定，使用类的全限定类名加一个后缀作为 beanName。
                else {
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    // Register an alias for the plain bean class name, if still possible,
                    // if the generator returned the class name plus a suffix.
                    // This is expected for Spring 1.2/2.0 backwards compatibility.
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                        beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Neither XML 'id' nor 'name' specified - " +
                                 "using generated bean name [" + beanName + "]");
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        // 返回一个 BeanDefinition 持有器。
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }

    return null;
}

/**
 * org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
 */
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, @Nullable BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));

    String className = null;
    // 解析 class 属性。
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    String parent = null;
    // 解析 parent 属性。
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    try {
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        // 解析并获取 description。
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        // 解析元数据。
        parseMetaElements(ele, bd);
        // 解析 lookup-method 属性。
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // 解析 replaced-method 属性。
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        // 解析构造器参数。
        parseConstructorArgElements(ele, bd);
        // 解析 property 子元素。
        parsePropertyElements(ele, bd);
        // 解析 qualifier 子元素。
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

/**
 * org.springframework.beans.factory.support.BeanDefinitionReaderUtils
 */
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    // 获取 BeanDefinitionHolder 中的 BeanName。
    String beanName = definitionHolder.getBeanName();
    // 往 BeanFactory 中注册 BeanDefinition。
    // BeanFactory 维护一个 beanDefinitionMap，是一个ConcurrentHashMap，存放 BeanName 与 BeanDefinition 的映射关系。
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            // 往 BeanFactory 中注册该 Bean 的别名。
            registry.registerAlias(beanName, alias);
        }
    }
}
```

