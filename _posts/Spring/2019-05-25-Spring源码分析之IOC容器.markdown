---
layout: post
title: Spring源码分析之IOC容器
date: 2019-05-25
comments: true 
tags: 学习笔记
summary: 本文是Spring Framework 5.1.7.RELEASE源码分析的第一章——IOC容器，主要围绕Spring是如何根据配置文件进行加载、注册并使用BEAN的。
---
### 功能分析

1. 配置文件加载：对XML信息进行验证，并根据XML配置内容进行Bean信息的加载
2. 根据Bean信息进行实例化，并注入相应的依赖

### 源码分析

#### ClassPathXmlApplicationContext

##### 继承关系图

![ClassPathXmlApplicationContext](../../images/blog/Spring/IOC/ClassPathXmlApplicationContext.png)

ClassPathXml：说明此类可加载ClassPath的XML文件（通过ResourceLoader接口获取文件加载 的能力）

ApplicationContext：是一个应用的上下文

##### 接口继承分析

- `BeanFactory`: Bean工厂，用于生产Bean（工厂模式）

```java
public interface BeanFactory {

	/**
	 * 引用，用于获取BeanFactory本身
	 */
	String FACTORY_BEAN_PREFIX = "&";


	/**
	 * 通过名称获取Bean
	 */
	Object getBean(String name) throws BeansException;

	/**
	 * 通过名称获取Bean，并要求是指定类型，不然抛出异常
	 */
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	/**
	 * 通过名称和指定的构造参数（或者工厂方法参数）获取Bean
	 */
	Object getBean(String name, Object... args) throws BeansException;

	/**
	 * 通过类型获取Bean
	 */
	<T> T getBean(Class<T> requiredType) throws BeansException;

	/**
	 * 通过类型和指定的构造参数（或者工厂方法参数）获取Bean
	 */
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	/**
	 * 获取指定类型的对象提供者（在隐性注入时有用，如构造函数Bean参数隐性注入）
	 */
	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	/**
	 * 获取通过ResolvableType参数指定的对象提供者
	 */
	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	/**
	 * 通过名称判断容器中是否包含Bean
	 */
	boolean containsBean(String name);

	/**
	 * 通过名称判断该Bean是否单例类型
	 */
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	/**
	 * 通过名称判断该Bean是否原型类型
	 */
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	/**
	 * 判断指定Bean是否为指定的类型
	 */
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	/**
	 * 判断指定Bean是否为指定的类型
	 */
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	/**
	 * 获取指定Bean的类型
	 */
	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	/**
	 * 获取指定Bean的别名
	 */
	String[] getAliases(String name);

}

```

  * `ListableBeanFactory`: 继承`BeanFactory`，使其具有枚举所有Bean的能力，BeanFactory只能通过名字或类型一个个取出来

```java
public interface ListableBeanFactory extends BeanFactory {

    /**
    	 * 通过beanName判断是否存在Bean定义
    	 */
    boolean containsBeanDefinition(String beanName);

    /**
    	 * 返回工厂中有多少个Bean定义
    	 */
    int getBeanDefinitionCount();

    /**
    	 * 返回工厂中所有Bean定义的名字
    	 */
    String[] getBeanDefinitionNames();

    /**
    	 * 返回匹配指定类型的Bean名字
    	 */
    String[] getBeanNamesForType(ResolvableType type);

    /**
    	 * 返回匹配指定类型的Bean名字
    	 */
    String[] getBeanNamesForType(@Nullable Class<?> type);

    /**
    	 * 返回匹配指定类型的Bean名字
    	 */
    String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);

    /**
    	 * 获取指定类型的Beans实例
    	 */
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;

    /**
    	 * 获取指定类型的Beans实例
    	 */
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
        throws BeansException;

    /**
    	 * 返回标记为指定注解类型的所有Bean名字
    	 */
    String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);

    /**
    	 * 返回标记为指定注解类型的所有Bean名字及实例
    	 */
    Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;

    /**
    	 * 查找指定beanName的bean是否具有指定的注解类型
    	 */
    @Nullable
    <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
        throws NoSuchBeanDefinitionException;

}
```

  * `HierarchicalBeanFactory`：继承`BeanFactory`，使其有层级功能

```java
public interface HierarchicalBeanFactory extends BeanFactory {

    /**
    	 * 返回父Bean工厂
    	 */
    @Nullable
    BeanFactory getParentBeanFactory();

    /**
    	 * 判断本地Bean工厂中是否包含指定name的，忽略祖先Bean工厂中的bean
    	 * @see BeanFactory#containsBean
    	 */
    boolean containsLocalBean(String name);

}
```




- `EnvironmentCapable`：提供获取`Environment`的能力

```java
public interface EnvironmentCapable {

	/**
	 * 返回该对象中关联的Environment.
	 */
	Environment getEnvironment();
}
```



* `ApplicationEventPublisher`：事件发布接口（观察者模式，发布-监听）

```java
public interface ApplicationEventPublisher {

	/**
	 * 发布事件通知到所有注册到此应用且支持该消息类型的listener中
	 */
	default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}

	/**
	 * 发布事件通知到所有注册到此应用且支持该消息类型的listener中
	 */
	void publishEvent(Object event);

}
```



* `ResourceLoader`：资源加载接口（策略模式：不同的策略加载不同的类型的资源，比如加载classpath或者文件系统资源）

```java
public interface ResourceLoader {

	/** 
	 * 从classpath中加载文件的URL前缀: "classpath:". 
	 */
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;

	/**
	 * 返回指定位置中的Resource（Spring中所有的资源都被抽象成了Resource类）
	 */
	Resource getResource(String location);

	/**
	 * 返回资源加载使用的ClassLoader，当客户端需要使用ClassLoader的时候能够与ResourceLoader保持一致，而不是依赖于当前线程上下文中的ClassLoader（可能不一致）
	 */
	@Nullable
	ClassLoader getClassLoader();

}
```

* `ResourcePatternResolver`：继承`ResourceLoader`接口，提供模式匹配路径解析功能

```java
public interface ResourcePatternResolver extends ResourceLoader {

	/**
	 * 查询classpath下所有匹配的资源文件的URL前缀（包括jar中的资源）
	 */
	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	/**
	 * 返回指定模式路径中的所有Resource
	 */
	Resource[] getResources(String locationPattern) throws IOException;

}
```



* `MessageSource`：解析参数化、国际化的消息（策略模式接口）

```java
public interface MessageSource {

	/**
	 * 尝试通过code去解析国际化消息，如果没有找到则返回默认消息
	 */
	@Nullable
	String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale);

	/**
	 * 尝试通过code去解析国际化消息，如果没有找到就抛出异常
	 */
	String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;

	/**
	 * 尝试通过MessageSourceResolvable去解析国际化消息，如果没有找到就抛出异常
	 */
	String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;

}

```



* `ApplicationContext`：继承了`EnvironmentCapable`，`ListableBeanFactory`，`HierarchicalBeanFactory`，`MessageSource`，`ApplicationEventPublisher`和`ResourcePatternResolver`，并提供配置功能

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {

	/**
	 * 获取当前应用上下文的ID
	 */
	@Nullable
	String getId();

	/**
	 * 获取当前上下文所在应用的名称
	 */
	String getApplicationName();

	/**
	 * 返回当前上下文用于显示的名字
	 */
	String getDisplayName();

	/**
	 * 返回当前上下文被初始加载的开始时间
	 */
	long getStartupDate();

	/**
	 * 返回父上下文，没有则返回null
	 */
	@Nullable
	ApplicationContext getParent();

	/**
	 * 返回AutowireCapableBeanFactory，一般不会在应用代码中直接使用
	 * AutowireCapableBeanFactory继承自BeanFactory，为工厂提供Autowire注入功能
	 */
	AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;

}

```



* `Lifecycle`：提供生命周期管理功能

```java
public interface Lifecycle {

	/**
	 * 开始
	 */
	void start();

	/**
	 * 结束
	 */
	void stop();

	/**
	 * 是否在运行
	 */
	boolean isRunning();

}

```



* `ConfigurableApplicationContext`：Spring SPI（服务提供接口，绝大多数情况下应用上下文都应该实现这个接口）

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {

	/**
	 * 资源文件可以使用的分隔符
	 */
	String CONFIG_LOCATION_DELIMITERS = ",; \t\n";

	/**
	 * 类型转换Bean的名字，如果不提供则用默认的转换类
	 */
	String CONVERSION_SERVICE_BEAN_NAME = "conversionService";

	/**
	 * LoadTimeWeaver(代码织入)bean的名字
	 */
	String LOAD_TIME_WEAVER_BEAN_NAME = "loadTimeWeaver";

	/**
	 * 工厂中environment bean的名字
	 */
	String ENVIRONMENT_BEAN_NAME = "environment";

	/**
	 * 工厂中system 属性 bean的名字
	 */
	String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";

	/**
	 * 工厂中system environment bean的名字
	 */
	String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";


	/**
	 * 设置当前上下文的唯一ID
	 */
	void setId(String id);

	/**
	 * 设置父级上下文
	 */
	void setParent(@Nullable ApplicationContext parent);

	/**
	 * 设置上下文的environment
	 */
	void setEnvironment(ConfigurableEnvironment environment);

	/**
	 * 获取enviroment
	 */
	@Override
	ConfigurableEnvironment getEnvironment();

	/**
	 * 添加一个BeanFactoryPostProcessor
	 */
	void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

	/**
	 * 添加一个ApplicationListener用于在上下文事件增加自定义处理
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * 增加额外的协议解析器
	 */
	void addProtocolResolver(ProtocolResolver resolver);

	/**
	 * 加载或者刷新上下文配置等
	 */
	void refresh() throws BeansException, IllegalStateException;

	/**
	 * 注册一个JVM shutdown时的钩子
	 */
	void registerShutdownHook();

	/**
	 * 关闭上下文，释放资源
	 */
	@Override
	void close();

	/**
	 * 返回当前上下文是否活动状态
	 */
	boolean isActive();

	/**
	 * 获取上下文的Bean工厂
	 */
	ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;

}

```



* `Aware`：Spring通知的标记接口超类

```java
public interface Aware {

}
```

* `BeanNameAware`：实现此接口以接收Bean名字的回调

```java
public interface BeanNameAware extends Aware {

	/**
	 * 设置Bean的名字
	 */
	void setBeanName(String name);

}
```



* `InitializingBean`：实现以此接口以接收当bean的所有配置参数被加载完成时的回调

```java
public interface InitializingBean {

	/**
	 * 容器在bean的所有属性设置完成时就会调用该方法
	 */
	void afterPropertiesSet() throws Exception;

}
```



##### 接口实现类分析

* `DefaultResourceLoader`：`ResourceLoader`的默认实现类

```java
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");

    // 查看注册的解析器能否解析该路径（可以调用addProtocolResolver进行注册解析器）
    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }

    // 否则
    if (location.startsWith("/")) {
        // 加载/开头的资源
        return getResourceByPath(location);
    }
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        // 加载以classpath开头的文件
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // 尝试以URL的方式解析资源
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // 失败时默认以从文件路径寻找资源.
            return getResourceByPath(location);
        }
    }
}
```

* `AbstractApplicationContext`：`ApplicationContext`的抽象实现，使用模板方法设计模式，需要具体的子类去实现抽象方法。

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
	// ResourcePatternResolver
	// 通过PathMatchingResourcePatternResolver实现getResources方法
	@Override
	public Resource[] getResources(String locationPattern) throws IOException {
		return this.resourcePatternResolver.getResources(locationPattern);
	}
	
	protected ResourcePatternResolver getResourcePatternResolver() {
		return new PathMatchingResourcePatternResolver(this);
	}
	
	// BeanFacotry、ListableBeanFactory及HierarchicalBeanFactory中的方法代理给BeanFacotry，子类需提供相应的实现，并且需要考虑性能问题
	@Override
	public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
	
	// 初始化消息解析器，消息的解析代理给此消息解析器完成
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		// 首先寻找bean工厂中是否有注册自定义messageSource，如果有，则用工厂中的
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// 使用空的MessageSource，以处理消息解析
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}
	
	// 初始化消息发布器
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		// 寻找bean工厂中是否有applicationEventMulticaster
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
		// 没有自定义的则使用默认的
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
	
	// EnvironmentCapable接口实现
	@Override
	public ConfigurableEnvironment getEnvironment() {
		if (this.environment == null) {
			this.environment = createEnvironment();
		}
		return this.environment;
	}
	
	protected ConfigurableEnvironment createEnvironment() {
		return new StandardEnvironment();
	}
	
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备刷新，如设置启动时间、设置状态等
			prepareRefresh();

			// 获取工厂实例，由子类实现
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 对beanFacotry做一些预处理
			prepareBeanFactory(beanFactory);

			try {
				// 允许子类对beanFactory进预处理
				postProcessBeanFactory(beanFactory);

				// 调用bean工厂预处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册Bean预处理器
				registerBeanPostProcessors(beanFactory);

				// 初始化消息解析器
				initMessageSource();

				// 初始化事件发布器
				initApplicationEventMulticaster();

				// 模板方法，默认空实现，可用于初始化特殊的bean
				onRefresh();

				// 注册事件监听器
				registerListeners();

				// 初始化剩余的单例BEAN
				finishBeanFactoryInitialization(beanFactory);

				// 完成刷新，并发布刷新事件
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
	
	// 对beanFacotry做一些预处理
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 将工厂的classsLoader与上下文的设置成一样
		beanFactory.setBeanClassLoader(getClassLoader());
		// SPEL表达式解析
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 注册ApplicationContextAwareProcessor
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		// 设置忽略的bean依赖（由其它方式提供）
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// 设置bean工厂中实际没有注册，但实际上可以解析的依赖
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 注册ApplicationListenerDetector
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// 检测是否有LoadTimeWeaver
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 注册默认的environment bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
}
```

