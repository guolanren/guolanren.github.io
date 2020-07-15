---
title: 随谈-properties文件的读取乱码
description: SpringBoot 项目从 *.properties 中读取中文内容配置，出现了乱码。下面就针对读取乱码现象，随便谈一下......
categories: 
 - code
tags:
 - 经验
 - 源码
 - spring boot
 - spring boot 源码
---

------

## 前言

​	**SpringBoot** 项目从 `*.properties` 中读取中文内容配置，出现了乱码。下面就针对读取乱码现象，随便谈一下......

## IDEA 文件编码

​	有个习惯，每次新建项目都会先去设置文件编码。在 **IDEA** 中，`Ctrl + Alt + s` 打开设置面板，搜索 **encoding** ，在 **File Encoding** 一栏中，可以对当前项目下的文件编码进行全局设置（一般都设置 **UTF-8**）。

![文件编码设置](https://github.com/guolanren/gallery/blob/master/found/2020-07-09-%E9%9A%8F%E8%B0%88-properties%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%BB%E5%8F%96%E4%B9%B1%E7%A0%81/Setting.png?raw=true)

## Spring Boot 的读取方式

​	读取配置文件，需要使用与之对应的编码进行解码，才能正确读取到正确的值。所以，**Spring Boot** 读取配置文件的方式，就是是否会导致乱码的关键。

### 搜索位置（SearchLocations）

```java
/**
 * org.springframework.boot.context.config.ConfigFileApplicationListener
 */
private Set<String> getSearchLocations() {
   // 如果指定了 CONFIG_LOCATION_PROPERTY(spring.config.location)，返回对应的值作为搜索位置
   if (this.environment.containsProperty(CONFIG_LOCATION_PROPERTY)) {
      return getSearchLocations(CONFIG_LOCATION_PROPERTY);
   }
   // 如果没有设置 searchLocations，
   // 默认为 DEFAULT_SEARCH_LOCATIONS(classpath:/,classpath:/config/,file:./,file:./config/)。
   // 另外，如果指定了 CONFIG_ADDITIONAL_LOCATION_PROPERTY(spring.config.additional-location)，
   // 添加对应的值。
   Set<String> locations = getSearchLocations(CONFIG_ADDITIONAL_LOCATION_PROPERTY);
   locations.addAll(
         asResolvedSet(ConfigFileApplicationListener.this.searchLocations, DEFAULT_SEARCH_LOCATIONS));
   return locations;
}

/**
 * org.springframework.boot.context.config.ConfigFileApplicationListener
 */
private Set<String> getSearchLocations(String propertyName) {
    Set<String> locations = new LinkedHashSet<>();
    if (this.environment.containsProperty(propertyName)) {
        for (String path : asResolvedSet(this.environment.getProperty(propertyName), null)) {
            if (!path.contains("$")) {
                path = StringUtils.cleanPath(path);
                if (!ResourceUtils.isUrl(path)) {
                    path = ResourceUtils.FILE_URL_PREFIX + path;
                }
            }
            locations.add(path);
        }
    }
    return locations;
}
```

### 属性资源加载器（PropertySourceLoader）

​	**Spring Boot** 通过 `PropertySourceLoader` 根据它所支持的文件后缀，尝试加载配置文件，最终以 `PropertySource` 的形式呈现。

```java
/**
 * org.springframework.boot.context.config.ConfigFileApplicationListener$Loader
 */
private void load(String location, String name, Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
    if (!StringUtils.hasText(name)) {
		...
    }
    // 对已处理过的文件后缀，做过滤用
    Set<String> processed = new HashSet<>();
    // 遍历所有 PropertySourceLoader
    for (PropertySourceLoader loader : this.propertySourceLoaders) {
        // 遍历当前 PropertySourceLoader 所支持的文件后缀
        for (String fileExtension : loader.getFileExtensions()) {
            // 标记该文件后缀为已处理
            if (processed.add(fileExtension)) {
                // 对相应的文件后缀尝试加载
                loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory, consumer);
            }
        }
    }
}
```

​	这里的 `PropertySourceLoader` 来自于 `ConfigFileApplicationListener` 的内部类 `Loader`，该内部类只有一个构造器，并在此初始化 `PropertySourceLoader` 。

```java
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    this.environment = environment;
    this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
    this.resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader();
    // 使用 SpringFactoriesLoader 加载 PropertySourceLoader 类实例
    this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
                                                                     getClass().getClassLoader());
}

public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
	...
    // 使用指定的类加载器加载服务
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    ...
    List<T> result = new ArrayList<>(factoryNames.size());
    for (String factoryName : factoryNames) {
        result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
    }
    AnnotationAwareOrderComparator.sort(result);
    return result;
}

private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
    try {
        Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);
        // 如果所指定的 instanceClass 不是 PropertySourceLoader，报异常
        if (!factoryClass.isAssignableFrom(instanceClass)) {
            throw new IllegalArgumentException(
                "Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
        }
        // 使用反射实例化相应的类
        return (T) ReflectionUtils.accessibleConstructor(instanceClass).newInstance();
    }
    catch (Throwable ex) {
            throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), ex);
    }
}
```

​	`SpringFactoriesLoader` 会读取类路径下的所有 `spring.factories` 文件，找寻指定类的具体实现（服务）。

![spring.factories](https://github.com/guolanren/gallery/blob/master/found/2020-07-09-%E9%9A%8F%E8%B0%88-properties%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%BB%E5%8F%96%E4%B9%B1%E7%A0%81/Spring-Factories.png?raw=true)

### 加载器加载配置

​	`PropertiesPropertySourceLoader` 会使用 `OriginTrackedPropertiesLoader` 加载配置，并将返回的结果封装成 `PropertySource` 集合返回。

```java
/**
 * org.springframework.boot.env.PropertiesPropertySourceLoader
 */
public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
    // 加载配置
    Map<String, ?> properties = loadProperties(resource);
    if (properties.isEmpty()) {
        return Collections.emptyList();
    }
    // 将结果封装成 PropertySource 集合返回
    return Collections.singletonList(new OriginTrackedMapPropertySource(name, properties));
}

/**
 * org.springframework.boot.env.PropertiesPropertySourceLoader
 */
private Map<String, ?> loadProperties(Resource resource) throws IOException {
    String filename = resource.getFilename();
    // 如果是 xml 文件直接使用 PropertiesLoaderUtils 加载
    if (filename != null && filename.endsWith(XML_FILE_EXTENSION)) {
        return (Map) PropertiesLoaderUtils.loadProperties(resource);
    }
    // 使用 OriginTrackedPropertiesLoader 加载
    return new OriginTrackedPropertiesLoader(resource).load();
}
```

​	`OriginTrackedPropertiesLoader` 加载配置使用的是 `CharacterReader`

```java
/**
 * org.springframework.boot.env.OriginTrackedPropertiesLoader
 */
public Map<String, OriginTrackedValue> load(boolean expandLists) throws IOException {
    // 使用 CharacterReader 读取配置文件
    try (CharacterReader reader = new CharacterReader(this.resource)) {
        Map<String, OriginTrackedValue> result = new LinkedHashMap<>();
        StringBuilder buffer = new StringBuilder();
        while (reader.read()) {
            // 读取键
            String key = loadKey(buffer, reader).trim();
            if (expandLists && key.endsWith("[]")) {
                key = key.substring(0, key.length() - 2);
                int index = 0;
                do {
                    // 读取值
                    OriginTrackedValue value = loadValue(buffer, reader, true);
                    put(result, key + "[" + (index++) + "]", value);
                    if (!reader.isEndOfLine()) {
                        reader.read();
                    }
                }
                while (!reader.isEndOfLine());
            }
            else {
                OriginTrackedValue value = loadValue(buffer, reader, false);
                put(result, key, value);
            }
        }
        return result;
    }
}
```

​	下图是 `CharacterReader` 的继承结构，可以看出，其最终将读取的任务交由 `java.io.Reader` 处理。并且根据 `debug` 显示的对象信息，`java.io.Reader` 使用的字符集是 **ISO-8859-1**，而这就是导致乱码的罪魁祸首。

![CharacterReader](https://github.com/guolanren/gallery/blob/master/found/2020-07-09-%E9%9A%8F%E8%B0%88-properties%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%BB%E5%8F%96%E4%B9%B1%E7%A0%81/CharacterReader.png?raw=true)

![debug](https://github.com/guolanren/gallery/blob/master/found/2020-07-09-%E9%9A%8F%E8%B0%88-properties%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%BB%E5%8F%96%E4%B9%B1%E7%A0%81/debug.png?raw=true)

## IDEA 勾选 Transparent native-to-ascii conversion

​	网上给出解决方法，大多都是勾选 **IDEA Settings** 的 `Transparent native-to-ascii conversion`（上述的 **Settings** 面板有）。

![IDEA-Transparent-Native-To-ASCII-Conversion](https://github.com/guolanren/gallery/blob/master/found/2020-07-09-%E9%9A%8F%E8%B0%88-properties%E6%96%87%E4%BB%B6%E7%9A%84%E8%AF%BB%E5%8F%96%E4%B9%B1%E7%A0%81/IDEA-Transparent-Native-To-ASCII-Conversion.png?raw=true)

​	根据官方文档解释，它会将非 `ISO 8859-1` 字符转义。但是在 **IDEA** 中，你仍能看到你所编辑的内容，如果使用普通的编辑器打开，会发现需要转义的字符已转义。

​	再根据上述的 `properties` 文件的加载过程，最终不会产生乱码。

## yml/yaml 文件读取

​	之前 `properties` 文件的方式走不通，临时采用 `yml` 的方式解决了问题。在配置文件的加载过程中，我们可以看到 `PropertySourceLoader` 除了有 `PropertiesPropertySourceLoader` 还有一个 `YamlPropertySourceLoader`，所以也顺便看看 `yml` 文件又是如何加载的。

### Spring Boot 启动读取

​	直到使用 `YamlPropertySourceLoader` 加载，之前的步骤跟 `properties` 文件方式是一致的，这里就不赘述。

```java
/**
 * org.springframework.boot.env.YamlPropertySourceLoader
 */
public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
    if (!ClassUtils.isPresent("org.yaml.snakeyaml.Yaml", null)) {
        throw new IllegalStateException(
            "Attempted to load " + name + " but snakeyaml was not found on the classpath");
    }
    // 使用 OriginTrackedYamlLoader 加载
    List<Map<String, Object>> loaded = new OriginTrackedYamlLoader(resource).load();
    if (loaded.isEmpty()) {
        return Collections.emptyList();
    }
    List<PropertySource<?>> propertySources = new ArrayList<>(loaded.size());
    for (int i = 0; i < loaded.size(); i++) {
        String documentNumber = (loaded.size() != 1) ? " (document #" + i + ")" : "";
        propertySources.add(new OriginTrackedMapPropertySource(name + documentNumber, loaded.get(i)));
    }
    return propertySources;
}

/**
 * org.springframework.boot.env.OriginTrackedYamlLoader
 */
public List<Map<String, Object>> load() {
    final List<Map<String, Object>> result = new ArrayList<>();
    process((properties, map) -> result.add(getFlattenedMap(map)));
    return result;
}

/**
 * org.springframework.beans.factory.config.YamlProcessor
 */
protected void process(MatchCallback callback) {
    Yaml yaml = createYaml();
    // 迭代每个资源进行处理
    for (Resource resource : this.resources) {
        // 使用 Yaml 处理资源
        boolean found = process(callback, yaml, resource);
        if (this.resolutionMethod == ResolutionMethod.FIRST_FOUND && found) {
            return;
        }
    }
}

/**
 * org.springframework.beans.factory.config.YamlProcessor
 */
private boolean process(MatchCallback callback, Yaml yaml, Resource resource) {
    int count = 0;
    try {
        ...
        // 使用 UnicodeReader 读取资源
        try (Reader reader = new UnicodeReader(resource.getInputStream())) {
            ...
        }
   ...
}

/**
 * org.yaml.snakeyaml.reader.UnicodeReader
 */
protected void init() throws IOException {
    // UnicodeReader 每次 read 时都会先执行 init() 方法初始化 internalIn2
    // 如果已经初始化过，直接返回
    if (internalIn2 != null)
        return;

    // 读取文件的 BOM 标记，来决定使用哪一种 Unicode 编码
    Charset encoding;
    byte bom[] = new byte[BOM_SIZE];
    int n, unread;
    n = internalIn.read(bom, 0, bom.length);

    if ((bom[0] == (byte) 0xEF) && (bom[1] == (byte) 0xBB) && (bom[2] == (byte) 0xBF)) {
        encoding = UTF8;
        unread = n - 3;
    } else if ((bom[0] == (byte) 0xFE) && (bom[1] == (byte) 0xFF)) {
        encoding = UTF16BE;
        unread = n - 2;
    } else if ((bom[0] == (byte) 0xFF) && (bom[1] == (byte) 0xFE)) {
        encoding = UTF16LE;
        unread = n - 2;
    } else {
        // Unicode BOM mark not found, unread all bytes
        // 如果没有 BOM 标记，默认使用 UTF-8
        encoding = UTF8;
        unread = n;
    }

    if (unread > 0)
        internalIn.unread(bom, (n - unread), unread);

    // Use given encoding
    CharsetDecoder decoder = encoding.newDecoder().onUnmappableCharacter(
        CodingErrorAction.REPORT);
    internalIn2 = new InputStreamReader(internalIn, decoder);
}
```

​	与 `properties` 类似，`yml/yaml` 也有与之对应的 `OriginTrackedYamlLoader`。其底层使用 `UnicodeReader` 读取资源，而在 **read()** 调用中的第一件事就是使用 **init()** 初始化 **internalIn2**。其中根据 **BOM** 来判断资源该使用何种 **Unicode** 编码（**UTF8/UTF16BE/UTF16LE**）。如果没有 **BOM** ，则使用 **UTF-8**。所以，在我们使用 **UTF-8** 编码配置文件，**Spring Boot** 读取结果不会产生乱码。

### @PropertySource 读取

​	有时需要将多个配置项存储到一个对象中，一个一个使用 `@Value` 不太方便，`@PropertySource` 则可以将配置封装在一个对象中。

```java
@Component
@ConfigurationProperties(prefix = "aliyun.sms")
@PropertySource(value = "classpath:config/aliyun-sms.properties")
public class SmsProperties {

    private String accessKey;
    private String secret;
    private String sign;
    private String templateCode;

	// Getter、Setter...
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(PropertySources.class)
public @interface PropertySource {
	String name() default "";
	String[] value();
	boolean ignoreResourceNotFound() default false;
	String encoding() default "";
    // 默认会使用 DefaultPropertySourceFactory(Spring Boot 中的唯一实现) 读取指定的配置
	Class<? extends PropertySourceFactory> factory() default PropertySourceFactory.class;
}

public class DefaultPropertySourceFactory implements PropertySourceFactory {

	@Override
	public PropertySource<?> createPropertySource(@Nullable String name, EncodedResource resource) throws IOException {
         // 默认的 PropertySourceFactory 将会使用 ResourcePropertySource 构建并返回 PropertySource
		return (name != null ? new ResourcePropertySource(name, resource) : new ResourcePropertySource(resource));
	}

}

public ResourcePropertySource(String name, EncodedResource resource) throws IOException {
    // 构建 ResourcePropertySource 时，使用 PropertiesLoaderUtils 加载资源返回 java.util.Properties
    super(name, PropertiesLoaderUtils.loadProperties(resource));
    this.resourceName = getNameForResource(resource.getResource());
}
```

​	所以默认情况下，`@PropertySource` 不支持 `yml/yaml` 的读取。使用自定义的 `PropertySourceFactory`，重写 **createPropertySource()** 方法。

```java
public class YamlPropertyLoaderFactory implements PropertySourceFactory {
    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
        List<PropertySource<?>> sources = new YamlPropertySourceLoader().load(name, resource.getResource());
        return sources.get(0);
    }
}
```



