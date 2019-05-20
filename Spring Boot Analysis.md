# Spring Boot （2.1.4.RELEASE）

## 自定义类`main`方法启动

```java
public static void main(String[] args) {
    //启动spring boot。
    SpringApplication.run(DovelolApplication.class, args);
}

public static ConfigurableApplicationContext run(Class<?> primarySource,
                                                 String... args) {
    //执行SpringApplication类中的静态run方法。
    return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                 String[] args) {
    //执行SpringApplication类中的实例run方法。
    return new SpringApplication(primarySources).run(args);
}
```



## `SpringApplication`类构造方法

```java
public SpringApplication(Class<?>... primarySources) {
    //调用构造函数。
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources)  {
    //由上面this调用传入的是null。
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //Set<Class<?>> primarySources中只有一条DovelolApplication类对象数据。
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //推断web应用类型(REACTIVE|NONE|SERVLET) 一般我们的应用都是SERVLET类型。
    this.webApplicationType = WebApplicationType.deduceFromClasspath();  
    //创建META-INF/spring.factories中类型为ApplicationContextInitializer的所有实例，添加到initializers集合，此处getSpringFactoriesInstances为核心方法，如果想自己添加需要2步，1：创建自定义类并且实现ApplicationContextInitializer接口；2：创建spring.factories文件并且配置好刚刚自己定义的Initializer类，可以设置Order值来调整加载顺序。
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    //创建META-INF/spring.factories中类型为ApplicationListener的所有实例,添加到listeners集合。
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //根据new RuntimeException().getStackTrace()显示的调用堆栈链路，循环找出main方法所在类的类对象
    //本例中是DovelolApplication的Class对象。
    this.mainApplicationClass = deduceMainApplicationClass();
}
```



##  `SpringApplication`类中的run方法

```java
/**
 * Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {
    //创建代码运行时间统计的实例
    StopWatch stopWatch = new StopWatch();
    //开始统计，设置开始时间为当前系统毫秒值
    stopWatch.start();
    //创建context变量
    ConfigurableApplicationContext context = null;
    //创建异常集合
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    //设置headless属性为true，
    configureHeadlessProperty();
    //创建META-INF/spring.factories中类型为SpringApplicationRunListener的所有实例（其实只有一个EventPublishingRunListener），此实例将来可以用作驱动所有listener监听的事件，如下所示。
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //第一处listener的调用，执行listeners中唯一的一个EventPublishingRunListener实例的starting方法，然后调用成员变量SimpleApplicationEventMulticaster（initialMulticaster）的multicastEvent方法，其实就是调用类型为ApplicationListener监听器的onApplicationEvent方法。这里有个疑问是到底调用了具体哪些listener（10个中筛选了4个）？，那么是怎么筛选出来的？？？？？
    listeners.starting();
    try {
        //创建ApplicationArguments实例
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                args);
        //#1 创建StandardServletEnvironment实例，并且加载配置资源
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                applicationArguments);
        //设置spring.beaninfo.ignore的值，先从系统参数中获取，如果没有，再从spring配置中获取。具体作用不清楚。
        configureIgnoreBeanInfo(environment);
        //初始化banner标语，也就是spring加载的图画，可以创建banner.txt文件来实现自定义图案。
        Banner printedBanner = printBanner(environment);
        //#2 根据webApplicationType类型创建对应实例，如果是Servlet,那么实例为AnnotationConfigServletWebServerApplicationContext,并且初始化各种父类的实例，如：GenericApplicationContext中this.beanFactory = new DefaultListableBeanFactory()；关键类RootBeanDefinition也是在这个过程中创建的，并且会注册一下内置的processor。
        context = createApplicationContext();
        //创建FailureAnalyzers类型所有的实例，并且BeanFactoryAware实现类设置BeanFactory，EnvironmentAware实现类设置Environment。
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        //#3 上下文准备工作
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                    .logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}

//#1
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
    // Create and configure the environment 类型为：StandardServletEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 设置环境配置
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    //第二处listener的调用，调用EventPublishingRunListener中的environmentPrepared方法，只是订阅的event事件是ApplicationEnvironmentPreparedEvent。
    listeners.environmentPrepared(environment);
    //把StandardServletEnvironment和SpringApplication绑定到一起。
    bindToSpringApplication(environment);
    //转换给定的environment为指定类型。
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    //添加一个name为configurationProperties的配置。
    ConfigurationPropertySources.attach(environment);
    return environment;
}

//#2
protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            //类型为SERVLET
            switch (this.webApplicationType) {
                case SERVLET:
                    //类型是org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
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
    //#2-1 关键方法，反射创建AnnotationConfigServletWebServerApplicationContext实例，以下用简称
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

//#2-1 
public AnnotationConfigServletWebServerApplicationContext() {
    //#2-1-1 构造函数实例化，创建DefaultListableBeanFactory对象，创建StandardServletEnvironment对象，注册内置bean的rdb到beanFactory中。
    this.reader = new AnnotatedBeanDefinitionReader(this);
    //#2-1-2 创建ClassPathBeanDefinitionScanner扫描类，可以扫描所有的bean并且注册为rdb。
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

//父类实例化
public ServletWebServerApplicationContext() {
}

//父类实例化
public GenericWebApplicationContext() {
    //调用父类构造函数。
    super();
}

//父类实例化
public GenericApplicationContext() {
    //创建DefaultListableBeanFactory对象，并且赋值给ACSWSAC.beanFactory变量。
    this.beanFactory = new DefaultListableBeanFactory();
}

//#2-1-1 
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    //getOrCreateEnvironment(registry)返回StandardServletEnvironment对象,调用其它构造函数。
    this(registry, getOrCreateEnvironment(registry));
}

//#2-1-1
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    //赋值ACSWSAC
    this.registry = registry;
    //实例化一个ConditionEvaluator以及它的内部类ConditionContextImpl，作用目前不详。
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    //#2-1-1-1 关键流程，总结下就是注册内置的各种Bean的BeanDefinition，添加到beanFactory中。
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}

//#2-1-1-1 
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
    //同上
    registerAnnotationConfigProcessors(registry, null);
}

//#2-1-1-1
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {
    //获取DefaultListableBeanFactory对象。
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    //判断DefaultListableBeanFactory不为null。
    if (beanFactory != null) {
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            //设置比较器，用于bean的排序。
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            //设置自动装配的处理器,用来处理延迟加载（@Lazy）的bean。如果一个bean上有@Lazy注解，个人理解就是在实例化的时候返回一个代理类并没有真正去实例化。
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }
	//创建一个类型为BeanDefinitionHolder的set集合。bdh是拥有名称和别名的BeanDefinition的持有者。可以注册为内部bean的占位符。
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        //创建一个ConfigurationClassPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        //创建一个AutowiredAnnotationBeanPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        //判断javax.annotation.Resource能不能被加载，如果可以的话创建一个CommonAnnotationBeanPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
         //判断javax.persistence.EntityManagerFactory能不能被加载，如果可以的话创建一个org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor类型的rbd。       
		def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,                                          AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        //判断ACSWSAC如果不包含org.springframework.context.event.internalEventListenerProcessor，创建一个EventListenerMethodProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        //判断ACSWSAC如果不包含org.springframework.context.event.internalEventListenerFactory，创建一个EventListenerMethodProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        //设置配置源。
        def.setSource(source);
        //关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }
	//返回初始化好各种内置bean的rdb set集合。
    return beanDefs;
}

//#2-1-2
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean 				useDefaultFilters,Environment environment, @Nullable ResourceLoader resourceLoader) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    //赋值registry=ACSWSAC。
    this.registry = registry;
	//使用默认过滤器。
    if (useDefaultFilters) {
        //#2-1-2-1 注册默认的注解过滤器类
        registerDefaultFilters();
    }
    //设置environment对象。
    setEnvironment(environment);
    //#2-1-2-2
    setResourceLoader(resourceLoader);
}
	
	//#2-1-2-1
	@SuppressWarnings("unchecked")
	protected void registerDefaultFilters() {
        //list集合中添加Component类型的注解过滤器。
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		//获取当前类的ClassLoader，Launcher$AppClassLoader。
        ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
            //list集合中添加javax.annotation.ManagedBean类型的注解过滤器，如果没有这个类会catch住ClassNotFoundException 异常。
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
             //list集合中添加javax.inject.Named类型的注解过滤器。
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}

	//#2-1-2-2
	public void setResourceLoader(@Nullable ResourceLoader resourceLoader) {
        //赋值this.resourcePatternResolver = ACSWSAC。
		this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
        //创建CachingMetadataReaderFactory对象，目的不详。
		this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
        //增加spring.components配置，加速包扫描速度。
		this.componentsIndex = CandidateComponentsIndexLoader.loadIndex(this.resourcePatternResolver.getClassLoader());
	}

	//#4
	private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		//ACSWSAC设置环境参数为environment，并且AbstractApplicationContext，reader，scanner都重新设置。这里有个疑问，既然这边重新设置，那为啥在创建ACSWSAC的时候要创建environment对象，这两个对象并不一致。
        context.setEnvironment(environment);
        //#4-1
		postProcessApplicationContext(context);
        //执行前面加载的所有Initializer的initialize方法，比如自定义的DemoInitializer。[DemoInitializer, DelegatingApplicationContextInitializer, SharedMetadataReaderFactoryContextInitializer, ContextIdApplicationContextInitialize, ConfigurationWarningsApplicationContextInitializer, ServerPortInfoApplicationContextInitializer, ConditionEvaluationReportLoggingListener]
		applyInitializers(context);
        //调用类型为ApplicationContextInitializedEvent的监听器的onApplicationEvent方法。
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof DefaultListableBeanFactory) {
			((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
	}

	//#4-1 
	protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
        //如果不为null，将name和实例添加到beanFactory中。
		if (this.beanNameGenerator != null) {
			context.getBeanFactory().registerSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
					this.beanNameGenerator);
		}
        //如果不为null，设置resourceLoader和classLoader。
		if (this.resourceLoader != null) {
			if (context instanceof GenericApplicationContext) {
				((GenericApplicationContext) context)
						.setResourceLoader(this.resourceLoader);
			}
			if (context instanceof DefaultResourceLoader) {
				((DefaultResourceLoader) context)
						.setClassLoader(this.resourceLoader.getClassLoader());
			}
		}
		if (this.addConversionService) {
            //设置beanFactory的数据转换类。
			context.getBeanFactory().setConversionService(
					ApplicationConversionService.getSharedInstance());
		}
	}

```



##  `SpringApplication`类中的run方法

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 准备上下文
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 返回DefaultListableBeanFactory，设置serializationId=application
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 设置DefaultListableBeanFactory各种配置
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 上下文是AnnotationConfigApplicationContext时，没有重写该方法，所以无操作
            // 上下文是AnnotationConfigServletWebServerApplicationContext时，进入该
            // 重写方法，创建WebApplicationContextServletContextAwareProcessor实例
            // 并且添加到beanFactory的beanPostProcessors中
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



