# 一、启动流程分析

每一个SpringBoot工程都是使用run方法开始启动应用

```java
SpringApplication.run(xxx.class, args);
```

run方法会使用传入的class作为参数创建一个SpringApplication实例

```java
new SpringApplication(primarySources)
```



## 1 SpringApplication构造过程

### 1.推断应用类型

根据当前运行环境中相应的class是否存在来判断应用是什么类型，比如servlet应用。

```java
static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
        && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
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
```

### 2.加载spring.factories配置文件

从META_INF/spring.factories中加载所有的springFactories

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

spring-boot.jar下面的META_INF/spring.factories配置中包含以下类型的工厂：

```properties
  # 配置加载器，比如propeties配置
  org.springframework.boot.env.PropertySourceLoader
  
  # SpringApplication运行监听器
  org.springframework.boot.SpringApplicationRunListener
  
  # 发生启动错误时的回调接口，用于自定义错误信息
  org.springframework.boot.SpringBootExceptionReporter
  
  # 应用上下文被刷新前的回调接口，用于初始化Spring
  org.springframework.context.ApplicationContextInitializer
  
  # 应用事件监听器
  org.springframework.context.ApplicationListener
  
  # 允许应用订制化（在应用上下文被刷新之前）
  org.springframework.boot.env.EnvironmentPostProcessor
  
  # 错误分析器：用于分析错误，并向用户提供诊断信息
  org.springframework.boot.diagnostics.FailureAnalyzer
  
  # 错误分析报告：向用户提供错误分析报告
  org.springframework.boot.diagnostics.FailureAnalysisReporter
```

### 3.初始化ApplicationContextInitializer

从配置文件中读取并初始化ApplicationContextInitializer实例

```java
setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
```

### 4.初始化ApplicationListener

从配置文件中读取并初始化ApplicationListener实例

```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```



## 2 SpringApplication运行流程

### 1. 初始化SpringApplicationRunListener

从配置文件中读取并初始化SpringApplicationRunListener实例

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
```
```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
   Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
   return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
         SpringApplicationRunListener.class, types, this, args));
}
```

### 2. 发布starting通知

```java
// 向所有在构造阶段初始化的ApplicationListener发送开始通知
listeners.starting();
```

### 3. 环境准备

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
```

```java
private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
    	// 配置环境参数
		configureEnvironment(environment, applicationArguments.getSourceArgs());
    	// 发布environmentPrepared事件
		listeners.environmentPrepared(environment);
    	// 绑定environment和application
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```

### 4. 打印Banner

```java
// 在控制台打印Banner
Banner printedBanner = printBanner(environment);
```

```java
private Banner printBanner(ConfigurableEnvironment environment) {
		if (this.bannerMode == Banner.Mode.OFF) {
			return null;
		}
		ResourceLoader resourceLoader = (this.resourceLoader != null)
				? this.resourceLoader : new DefaultResourceLoader(getClassLoader());
		SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(
				resourceLoader, this.banner);
		if (this.bannerMode == Mode.LOG) {
            // 如果开启了log模式，就打印到日志
			return bannerPrinter.print(environment, this.mainApplicationClass, logger);
		}
    	// 默认打印到控制台
		return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
	}
```

```java
// 打印banner
public Banner print(Environment environment, Class<?> sourceClass, PrintStream out) {
    Banner banner = getBanner(environment);
    banner.printBanner(environment, sourceClass, out);
    return new PrintedBanner(banner, sourceClass);
}
```

```java
// 获取banner
private Banner getBanner(Environment environment) {
    Banners banners = new Banners();
    // 获取图片和文本banner，默认前缀为banner
    
    // 获取图片banner，支持"gif", "jpg", "png"后缀
    // 可通过配置spring.banner.image.location进行自定义配置
    banners.addIfNotNull(getImageBanner(environment));
    // 获取文本banner，默认名称为banner.txt
    // 可通过配置spring.banner.location进行自定义配置
    banners.addIfNotNull(getTextBanner(environment));
    if (banners.hasAtLeastOneBanner()) {
        return banners;
    }
    if (this.fallbackBanner != null) {
        return this.fallbackBanner;
    }
    return DEFAULT_BANNER;
}
```

### 5. 创建上下文

```java
context = createApplicationContext();
```

```java
// 根据当前的环境，创建相应的上下文，如比较常见的servlet上下文（典型WEB应用）
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Unable create a default ApplicationContext, "
                + "please specify an ApplicationContextClass",
                ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

```

### 6. 初始化SpringBootExceptionReporter

```java
exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
```

### 7. 准备上下文

```java
prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
```

```java
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    // 应用ApplicationContextInitializer进行初始化
    applyInitializers(context);
    // 发布contextPrepared事件
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 注册参数bean
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        // 注册banner bean
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 加载上下文资源，如bean定义
    load(context, sources.toArray(new Object[0]));
    // 发布contextLoaded事件
    listeners.contextLoaded(context);
}
```

### 8. 刷新上下文

```java
refreshContext(context);
```

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

### 9. 发布started事件

```java
listeners.started(context);
```

### 10. 调用runners

获取ApplicationRunner和CommandLineRunner接口的实现，并调用，二者的不同点在于一个传递的是ApplicationArguments，一个传递的是原来的参数args

```java
callRunners(context, applicationArguments);
```

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

### 11.  发布running事件

```java
listeners.running(context);
```



# 二、spring.factories分析

## 1 PropertySourceLoader

PropertySourceLoader实现了配置的加载，默认有两种实现，分别是PropertiesPropertySourceLoader和YamlPropertySourceLoader。

默认加载的配置名称前缀都是`application`，可以通过`spring.config.name`变量改变配置名称。

默认加载的路径是`classpath:/,classpath:/config/,file:./,file:./config/`，扫描顺序是反过来的，即优先扫描`file:./config/`，可以通过`spring.config.location`变量改变扫描路径，或者通过`spring.config.additional-location`变量增加额外的扫描路径。

* PropertiesPropertySourceLoader

  支持的文件后缀：

  ```java
  public String[] getFileExtensions() {
      return new String[] { "properties", "xml" };
  }
  ```

* YamlPropertySourceLoader

  支持的文件后缀：

  ```java
  public String[] getFileExtensions() {
      return new String[] { "yml", "yaml" };
  }
  ```

加载时机：`ConfigFileApplicationListener`收到`ApplicationEnvironmentPreparedEvent`事件

```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
        // properties在此位置加载
        onApplicationEnvironmentPreparedEvent(
            (ApplicationEnvironmentPreparedEvent) event);
    }
    if (event instanceof ApplicationPreparedEvent) {
        onApplicationPreparedEvent(event);
    }
}
```

```java
// 该函数会加载所有的EnvironmentPostProcessor，并调用postProcessEnvironment方法
// 其中包括properties加载，也包括环境变量，jvm变量等参数的加载
private void onApplicationEnvironmentPreparedEvent(
    ApplicationEnvironmentPreparedEvent event) {
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    // ConfigFileApplicationListener也实现了EnvironmentPostProcessor接口，所以也添加了this
    postProcessors.add(this);
    AnnotationAwareOrderComparator.sort(postProcessors);
    for (EnvironmentPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessEnvironment(event.getEnvironment(),
                                             event.getSpringApplication());
    }
}
```

如果需要自定义property加载，实现PropertySourceLoader接口即可，比如实现一个JSON格式的配置文件加载 。

```java
public class JsonPropertySourceLoader implements PropertySourceLoader {

    private static final String JSON_PROPERTIES_EXTENSION = ".jsonprop";

    @Override
    public String[] getFileExtensions() {
        return new String[] { "jsonprop" };
    }

    @Override
    public List<PropertySource<?>> load(String name, Resource resource) throws IOException {
        String filename = resource.getFilename();
        if (filename != null && filename.endsWith(JSON_PROPERTIES_EXTENSION)) {
            Properties properties = loadProperties(resource);

            if (properties.isEmpty()) {
                return Collections.emptyList();
            }
            return Collections
                    .singletonList(new OriginTrackedMapPropertySource(name, properties));
        }
        return new ArrayList<>();
    }

    /**
     * 加载JSON格式的配置文件，不支持内容嵌套
     * @param resource
     * @return
     * @throws IOException
     */
    private Properties loadProperties(Resource resource) throws IOException {
        InputStream input = resource.getInputStream();
        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, String> data = objectMapper.readValue(input, Map.class);

        Properties props = new Properties();
        data.forEach((key, value) -> {
            props.setProperty(key, value);
        });

        return props;
    }
}

```

如代码所示，后缀名为`jsonprop`，在`resource`目录下添加`application.jsonprop`配置文件。

```json
{
  "applicationName":"SpringBootDemo"
}
```

在`resource/META-INF/spring.factories`中（如果没有，则需先添加）注册PropertySourceLoader

```properties
// 注册相应的类名（需要带包名）
org.springframework.boot.env.PropertySourceLoader=xxx.xxx.JsonPropertySourceLoader
```

然后就可以在代码中注入相应的配置内容了，比如：

```java
@Value("${applicationName}")
private String applicationName;
```

## 2 SpringApplicationRunListener

默认注册实例为`EventPublishingRunListener`，用于向所有的`ApplicationListener`发布相应的事件，例如`ApplicationStartingEvent`事件：

```java
public void starting() {
    // 广播ApplicationStartingEvent事件
    this.initialMulticaster.multicastEvent(
        new ApplicationStartingEvent(this.application, this.args));
}
```

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    
    // 取出支持该事件的ApplicationListener，并依次发送通知
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            invokeListener(listener, event);
        }
    }
}
```

## 3 SpringBootExceptionReporter

```java
public boolean reportException(Throwable failure) {
    // analyzers由构造器初始化，加载所有的FailureAnalyzer
    // 循环调用所有的FailureAnalyzer进行分析当前的异常
    FailureAnalysis analysis = analyze(failure, this.analyzers);
    return report(analysis, this.classLoader);
}
```

```java
private boolean report(FailureAnalysis analysis, ClassLoader classLoader) {
    List<FailureAnalysisReporter> reporters = SpringFactoriesLoader
        .loadFactories(FailureAnalysisReporter.class, classLoader);
    if (analysis == null || reporters.isEmpty()) {
        return false;
    }
    // 调用所有的FailureAnalysisReporter进行错误报告
    for (FailureAnalysisReporter reporter : reporters) {
        reporter.report(analysis);
    }
    return true;
}
```

## 4 ApplicationContextInitializer

在**准备上下文**阶段进行初始化调用

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

