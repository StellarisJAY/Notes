# SpringBoot自动装配原理

## @SpringBootApplication注解

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication
```

该注解包含了三个注解，@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan

其中@SpringBootConfiguration注解中包含了@Configuration，这个注解的作用是声明该类为配置类，我们可以理解为@SpringBootConfiguration = @Configuration。

@ComponentScan再熟悉不过了，就是让Spring去扫描与SpringBootApplication同级目录下的组件。

所以，自动装配最关键的还是@EnableAutoConfiguration这个注解。

## @EnableAutoConfiguration

```java
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration
```

在该注解中有一个@Import，它导入了一个叫EnableAutoConfigurationImportSelector的类，这个类就是自动装配的关键。

可以发现这个类实际上是继承了AutoConfigurationImportSelector这个类。

在AutoConfigurationImportSelector中可以找到一个叫getCandidateConfigurations的方法，它返回一个String列表，也就是获取候选的配置类类名

```java
	/**
	 * 返回可以考虑自动装配的类的类名
	 * Return the auto-configuration class names that should be considered. By default
	 * this method will load candidates using {@link SpringFactoriesLoader} with
	 * {@link #getSpringFactoriesLoaderFactoryClass()}.
	 * @param metadata the source metadata
	 * @param attributes the {@link #getAttributes(AnnotationMetadata) annotation
	 * attributes}
	 * @return a list of candidate configurations
	 */
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

那么SpringBoot是从哪里获取到这些候选类的呢？

来看loadFactoryNames这个方法。

```java
	// Factories文件路径
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";	
	
	public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
		String factoryTypeName = factoryType.getName();
        // 调用loadSpringFactories，返回一个Map，从Map获取对应factoryType的类名列表
		return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
	}
	
	/**
	* 加载Factories，返回一个Map	
	*
	*/
	private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
		Map<String, List<String>> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		result = new HashMap<>();
		try {
            // 打开了某个文件
			Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
                // 加载一条记录的内容
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryTypeName = ((String) entry.getKey()).trim();
					String[] factoryImplementationNames =
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
					for (String factoryImplementationName : factoryImplementationNames) {
						result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
								.add(factoryImplementationName.trim());
					}
				}
			}

			// Replace all lists with unmodifiable lists containing unique elements
			result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
					.collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
			cache.put(classLoader, result);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
		return result;
	}
```

一大坨代码也没注释看着烦，所以只看关键部分。

首先loadFactorieNames是调用了loadSpringFactories方法来获取结果，loadSpringFactories中最关键的是它打开了**FACTORIES_RESOURCE_LOCATION**中的路径。

根据FACTORIES_RESOURCE_LOCATION，我们找到**META-INF/spring.factories**这个文件打开。


![spring.factories.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9de9adb24fb4bd6a83cdb445639956e~tplv-k3u1fbpfcp-watermark.image?)

可以发现这个文件里面有一大堆的全类名，这些类都是以XXXAutoConfiguration命名的。这其中有很多常见的配置类，比如Rabbit、ElasticSearch、jdbc等等。

看到这里也就清楚了，SpringBoot是从spring.factories这个文件中找到了所有候选的自动配置类。

但是又有一个问题，就是这里有这么多自动配置类，每次都把它们全部加载进来配置太浪费时间了。其实SpringBoot并不会全部配置。

拿RabbitMQ的RabbitAutoConfiguration来说

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })
@EnableConfigurationProperties(RabbitProperties.class)
@Import(RabbitAnnotationDrivenConfiguration.class)
public class RabbitAutoConfiguration {
    ...
}
```

可以看到这个类上除了@EnableConfigurationProperties来导入配置属性这些，还有@ConditionOnClass注解。

@ConditionOnXXX注解表示要在满足条件的情况下才进行配置，比如这里要在有RabbitTemplate和Channel这两个类的情况下才自动装配。

这也就是为什么spring.factories中有那么多类，最终SpringBoot却只会加载我们需要的类的原因了。



## 总结


![自动装配.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40c3fb6eb1b6420a89d1fb89e1ffda92~tplv-k3u1fbpfcp-watermark.image?)

1. @SpringBootApplication包括了@ComponentScan、@SpringBootConfiguration、@EnableAutoConfiguration，其中@EnalbeAutoConfiguration是自动装配的关键。
2. @EnableAutoConfiguration中有@Import，导入了AutoConfigurationImportSelector。
3. AutoConfigurationImportSelector中的getCandidateConfigurations会从META-INF/spring.factories加载类
4. 加载时会判断类的@ConditionONXXX是否成立，成立才加载

