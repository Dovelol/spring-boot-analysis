# Spring Boot （2.1.4.RELEASE）

## 自定义类`main`方法启动

```java
public static void main(String[] args) {
    // 启动spring boot。
    SpringApplication.run(DovelolApplication.class, args);
}

public static ConfigurableApplicationContext run(Class<?> primarySource,
                                                 String... args) {
    // 执行SpringApplication类中的静态run方法。
    return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
                                                 String[] args) {
    // 创建SpringApplication对象，并执行run方法。
    return new SpringApplication(primarySources).run(args);
}
```



## `SpringApplication`类构造方法

```java
public SpringApplication(Class<?>... primarySources) {
    // 调用构造函数。
    this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources)  {
    // 由上面this调用传入的是null。
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // Set<Class<?>> primarySources中只有一条DovelolApplication类对象数据。
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断web应用类型(REACTIVE|NONE|SERVLET) 一般我们的应用都是SERVLET类型。
    this.webApplicationType = WebApplicationType.deduceFromClasspath();  
    // 创建META-INF/spring.factories中类型为ApplicationContextInitializer的所有实例，添加到initializers集合，此处getSpringFactoriesInstances为核心方法，如果想自己添加需要2步，1：创建自定义类并且实现ApplicationContextInitializer接口；2：创建spring.factories文件并且配置好刚刚自己定义的Initializer类，可以设置Order值来调整加载顺序。
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    // 创建META-INF/spring.factories中类型为ApplicationListener的所有实例,添加到listeners集合。
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 根据new RuntimeException().getStackTrace()显示的调用堆栈链路，循环找出main方法所在类的类对象，本例中是DovelolApplication的Class对象。
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
    // 创建代码运行时间统计的实例
    StopWatch stopWatch = new StopWatch();
    // 开始统计，设置开始时间为当前系统毫秒值
    stopWatch.start();
    // 创建context变量
    ConfigurableApplicationContext context = null;
    // 创建异常集合
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 设置headless属性为true。
    configureHeadlessProperty();
    // 创建META-INF/spring.factories中类型为SpringApplicationRunListener的所有实例（其实只有一个EventPublishingRunListener），在EventPublishingRunListener构造器中先创建SimpleApplicationEventMulticaster对象，然后从SpringApplication对象中获取到的所有的ApplicationListener（listener）添加到SimpleApplicationEventMulticaster的ListenerRetriever（defaultRetriever）对象中，后续所有listener的过滤都是从这个ListenerRetriever中获取的；然后创建SpringApplicationRunListeners对象，把刚刚SpringApplicationRunListener的所有实例添加到listeners属性中。简单点说结构就是SpringApplicationRunListeners->EventPublishingRunListener->SimpleApplicationEventMulticaster，最后调用的全是都是SimpleApplicationEventMulticaster中的方法。
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 第一处listener的调用，执行listeners中唯一的一个EventPublishingRunListener实例的starting方法，然后调用成员变量SimpleApplicationEventMulticaster（initialMulticaster）的multicastEvent方法，其实就是调用类型为ApplicationListener监听器的onApplicationEvent方法。这里有个疑问是到底调用了具体哪些listener（10个中筛选了4个）？，那么是怎么筛选出来的？？？？？回答上面的问题，是通过筛选对应的ApplicationEvent事件来找出要求的ApplicationListener，然后依次调用onApplicationEvent方法。此处有2个接口，分别是GenericApplicationListener（@since 4.2）和SmartApplicationListener（@since 3.0），这两个接口的区别就是supportsEventType方法的入参类型不一样，新版本后由GenericApplicationListener替代。
    listeners.starting();
    try {
        // 创建ApplicationArguments实例
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                args);
        // #1 创建StandardServletEnvironment实例，并且加载配置资源
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                applicationArguments);
        // 设置spring.beaninfo.ignore的值，先从系统参数中获取，如果没有，再从spring配置中获取。具体作用不清楚。
        configureIgnoreBeanInfo(environment);
        // 初始化banner标语，也就是spring加载的图画，可以创建banner.txt文件来实现自定义图案。
        Banner printedBanner = printBanner(environment);
        // #2 根据webApplicationType类型创建对应实例，如果是Servlet,那么实例为AnnotationConfigServletWebServerApplicationContext,并且初始化各种父类的实例，如：GenericApplicationContext中this.beanFactory = new DefaultListableBeanFactory()；关键类RootBeanDefinition也是在这个过程中创建的，并且会注册一下内置的processor。
        context = createApplicationContext();
        // 创建FailureAnalyzers类型所有的实例，并且BeanFactoryAware实现类设置BeanFactory，EnvironmentAware实现类设置Environment。
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // #3 上下文准备工作，主要是设置环境变量，执行所有Initializer的方法，还有就是注册一些内置默认的bean。
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);
        // #4 初始化spring context。
        refreshContext(context);
        // 初始化完成后的操作。
        afterRefresh(context, applicationArguments);
        // 结束统计。
        stopWatch.stop();
        // 打印工程启动时间。
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                    .logStarted(getApplicationLog(), stopWatch);
        }
        // 发布ApplicationStartedEvent类型的事件。
        listeners.started(context);
        // 调用ApplicationRunner和CommandLineRunner实现类的run方法，可以实现这两个接口来完成启动后的操作，传入的是ApplicationArguments这个参数。
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

// #1
private ConfigurableEnvironment prepareEnvironment(
    SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
    // Create and configure the environment 创建StandardServletEnvironment对象，并且将系统配置（systemProperties）和系统环境变量（systemEnvironment）添加到propertySources属性中。
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 设置环境配置包括了从args中获取的参数还有additionalProfiles种的配置。
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 第二处listener的调用，只针对订阅ApplicationEnvironmentPreparedEvent类型的event事件，依次调用其onApplicationEvent方法。
    listeners.environmentPrepared(environment);
    // 把StandardServletEnvironment和SpringApplication绑定到一起。（具体操作没有看懂）
    bindToSpringApplication(environment);
    // 转换给定的environment为指定类型。
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
            .convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    // 添加一个name为configurationProperties的配置。
    ConfigurationPropertySources.attach(environment);
    return environment;
}

//#2
protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 类型为SERVLET
            switch (this.webApplicationType) {
                case SERVLET:
                    // 类型是org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
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
    // #2-1 关键方法，反射创建AnnotationConfigServletWebServerApplicationContext实例，以下用简称。
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

// #2-1 
public AnnotationConfigServletWebServerApplicationContext() {
    // #2-1-1 构造函数实例化，创建DefaultListableBeanFactory对象，创建StandardServletEnvironment对象，注册内置bean的rdb到beanFactory中。
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // #2-1-2 创建ClassPathBeanDefinitionScanner扫描类，可以扫描所有的bean并且注册为rdb。
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

// 父类实例化
public ServletWebServerApplicationContext() {
}

// 父类实例化
public GenericWebApplicationContext() {
    // 调用父类构造函数。
    super();
}

// 父类实例化
public GenericApplicationContext() {
    // 创建DefaultListableBeanFactory对象，并且赋值给ACSWSAC.beanFactory变量。
    this.beanFactory = new DefaultListableBeanFactory();
}

// #2-1-1 
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    // getOrCreateEnvironment(registry)返回StandardServletEnvironment对象,调用其它构造函数。
    this(registry, getOrCreateEnvironment(registry));
}

//#2-1-1
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    // 赋值ACSWSAC
    this.registry = registry;
    // 实例化一个ConditionEvaluator以及它的内部类ConditionContextImpl，作用目前不详。
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    // #2-1-1-1 关键流程，总结下就是注册内置的各种Bean的BeanDefinition，添加到beanFactory中。
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}

// #2-1-1-1 
public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
    // 同上
    registerAnnotationConfigProcessors(registry, null);
}

// #2-1-1-1
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {
    // 获取DefaultListableBeanFactory对象。
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    // 判断DefaultListableBeanFactory不为null。
    if (beanFactory != null) {
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            // 设置比较器，用于bean的排序。
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            // 设置自动装配的处理器,用来处理延迟加载（@Lazy）的bean。如果一个bean上有@Lazy注解，个人理解就是在实例化的时候返回一个代理类并没有真正去实例化。
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }
	// 创建一个类型为BeanDefinitionHolder的set集合。bdh是拥有名称和别名的BeanDefinition的持有者。可以注册为内部bean的占位符。
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        // 创建一个ConfigurationClassPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        // 创建一个AutowiredAnnotationBeanPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        // 判断javax.annotation.Resource能不能被加载，如果可以的话创建一个CommonAnnotationBeanPostProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
         // 判断javax.persistence.EntityManagerFactory能不能被加载，如果可以的话创建一个org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor类型的rbd。       
		def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,                                          AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        // 判断ACSWSAC如果不包含org.springframework.context.event.internalEventListenerProcessor，创建一个EventListenerMethodProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        // 判断ACSWSAC如果不包含org.springframework.context.event.internalEventListenerFactory，创建一个EventListenerMethodProcessor类型的rbd。
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        // 设置配置源。
        def.setSource(source);
        // 关键流程，将这个bean注册到beanFactory中，并且组装成dbh对象添加到set里面。
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }
	// 返回初始化好各种内置bean的rdb set集合。
    return beanDefs;
}

//#2-1-2
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean 				useDefaultFilters,Environment environment, @Nullable ResourceLoader resourceLoader) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    // 赋值registry=ACSWSAC。
    this.registry = registry;
	// 使用默认过滤器。
    if (useDefaultFilters) {
        // #2-1-2-1 注册默认的注解过滤器类
        registerDefaultFilters();
    }
    // 设置environment对象。
    setEnvironment(environment);
    //#2-1-2-2
    setResourceLoader(resourceLoader);
}
	
	//#2-1-2-1
@SuppressWarnings("unchecked")
protected void registerDefaultFilters() {
    // list集合中添加Component类型的注解过滤器。
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    // 获取当前类的ClassLoader，Launcher$AppClassLoader。
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
    try {
        // list集合中添加javax.annotation.ManagedBean类型的注解过滤器，如果没有这个类会catch住ClassNotFoundException 异常。
        this.includeFilters.add(new AnnotationTypeFilter(
            ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
        logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
    }
    try {
        // list集合中添加javax.inject.Named类型的注解过滤器。
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
    // 赋值this.resourcePatternResolver = ACSWSAC。
    this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
    // 创建CachingMetadataReaderFactory对象，目的不详。
    this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
    // 增加spring.components配置，加速包扫描速度。
    this.componentsIndex = CandidateComponentsIndexLoader.loadIndex(this.resourcePatternResolver.getClassLoader());
}

//#3
private void prepareContext(ConfigurableApplicationContext context,
                            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
                            ApplicationArguments applicationArguments, Banner printedBanner) {
    // ACSWSAC设置环境参数为environment，并且AbstractApplicationContext，reader，scanner都重新设置。这里有个疑问，既然这边重新设置，那为啥在创建ACSWSAC的时候要创建environment对象，这两个对象并不一致。这里是重新new了一个包含environment属性的ConditionEvaluator对象，讲run方法中创建的environment对象设置进去。
    context.setEnvironment(environment);
    // #3-1
    postProcessApplicationContext(context);
    // 执行前面加载的所有Initializer的initialize方法，比如自定义的DemoInitializer。[DemoInitializer, DelegatingApplicationContextInitializer, SharedMetadataReaderFactoryContextInitializer, ContextIdApplicationContextInitialize, ConfigurationWarningsApplicationContextInitializer, ServerPortInfoApplicationContextInitializer, ConditionEvaluationReportLoggingListener]
    applyInitializers(context);
    // 调用类型为ApplicationContextInitializedEvent的监听器的onApplicationEvent方法。
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        //如果context是根结构，那么打印启动信息，比如ApplicationName、pid、context信息等。
        logStartupInfo(context.getParent() == null);
        //打印启动应用的环境属性信息。
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    //获取到context的beanFactory。
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    //将applicationArguments注册到beanFactory中。
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        //如果启动标志类不为null，也注册到beanFactory中。
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    //判断beanFactory的类型。
    if (beanFactory instanceof DefaultListableBeanFactory) {
        //设置allowBeanDefinitionOverriding。
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    //获取所有的sources。
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    //将自定义启动类注册到beanFactory中。
    load(context, sources.toArray(new Object[0]));
    //调用监听器的onApplicationEvent方法，事件类型是ApplicationPreparedEvent。
    listeners.contextLoaded(context);
}

//#3-1 
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

//#4 
private void refreshContext(ConfigurableApplicationContext context) {
    // #4-1 启动上下文context 
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            // 注册一个钩子可以在jvm销毁的时候触发doClose操作，发送一个ContextClosedEvent类型的事件，销毁注册的bean，关闭beanFactory，也就是设置setSerializationId=null，关闭tomcat相关的容器。
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

//#4-1 连续调用
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}

//#4-1 
public final void refresh() throws BeansException, IllegalStateException {
    try {
        //最终调用到SpringApplication类中的refresh方法
        super.refresh();
    }
    catch (RuntimeException ex) {
        stopAndReleaseWebServer();
        throw ex;
    }
}


```



##  `SpringApplication`类中的refresh方法

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    // 这里加锁，主要是防止并发的启动context。
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        // 准备上下文，主要是设置启动时间、活跃状态，还有初始化一些配置。
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 返回DefaultListableBeanFactory，设置serializationId=application
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // #1 设置DefaultListableBeanFactory各种配置
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.上下文是AnnotationConfigApplicationContext时，没有重写该方法，所以无操作，上下文是AnnotationConfigServletWebServerApplicationContext时，进入该重写方法，创建WebApplicationContextServletContextAwareProcessor实例并且添加到beanFactory的beanPostProcessors中
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 代码太多了，看了半天，基本上是这样的，扫描所有的@Component注解的类，然后生成BeanDefinition并且注册到beanFactory中去。
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 找出所有类型为BeanPostProcessor的bean,然后创建这些bean并且全部注册到beanFactory中，可以说这一步就是BeanPostProcessor实例化然后注册的过程。
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 初始化消息源,判断beanFactory中是否有名字为messageSource的bean，如果没有，新建DelegatingMessageSource类作为bean，如果有的话，从beanFactory中获取。
            initMessageSource();

            // Initialize event multicaster for this context.
            // 初始化ApplicationEventMulticaster，如果beanFactory中不存在的话实例化SimpleApplicationEventMulticaster，如果存在使用从beanFactory中获取的。
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // #2初始化内置的tomcat服务。
            onRefresh();

            // Check for listener beans and register them.
            // 没看懂啥操作，就是把一些listener添加到defaultRetriever.applicationListeners和defaultRetriever.applicationListenerBeans中去。
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // #3完成bean的创建。
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 发布ContextRefreshedEvent事件，对应的listener执行相应的处理。并且启动web容器webServer.start()，也就是说调用bind方法，正式开启端口监听是在这一部进行的，之后创建名称为http-nio-8080-exec-xxx的线程组，这个是之后接受请求后的工作线程。然后创建http-nio-8080-ClientPoller-0的线程。最后创建http-nio-8080-Acceptor-0的线程。
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

//#1 
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    // 设置ClassLoader
    beanFactory.setBeanClassLoader(getClassLoader());
    // 设置表达式处理器为StandardBeanExpressionResolver。
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 设置ResourceEditorRegistrar。
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    //beanPostProcessors集合中添加ApplicationContextAwareProcessor。
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //ignoredDependencyInterfaces集合中添加EnvironmentAware的class对象，作用是忽略依赖检查和注入。下同。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    //Map<Class<?>, Object> resolvableDependencies添加依赖类型和自动装配的值。
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    //beanPostProcessors集合中添加ApplicationListenerDetector处理器。
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    //判断beanFactory是否包含LoadTimeWeaver。如果有的话就添加对应的processor。
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    //判断beanFactory是否包含environment。
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        //注册单例的environment对象
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    //判断beanFactory是否包含systemProperties。
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        //注册单例的systemProperties对象
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    //判断beanFactory是否包含systemEnvironment。
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        //注册单例的systemEnvironment对象。
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}


// #2-1
@Override
protected void onRefresh() {
    // 初始化themeSource，如果beanFactory中不存在的话实例化ResourceBundleThemeSource，如果存在使用从beanFactory中获取的。
    super.onRefresh();
    try {
        // #2-1-1 创建web服务
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}

// #2-1-1
private void createWebServer() {
    // 获取WebServer对象。
    WebServer webServer = this.webServer;
    // 获取ServletContext对象。
    ServletContext servletContext = getServletContext();
    // 如果上述两个对象为null。
    if (webServer == null && servletContext == null) {
        // 获取ServletWebServerFactory对象，本例中用的是tomcat，所以实例化tomcatServletWebServerFactory对象。
        ServletWebServerFactory factory = getWebServerFactory();
        // #2-1-1-1
        this.webServer = factory.getWebServer(getSelfInitializer());
    }
    else if (servletContext != null) {
        try {
            getSelfInitializer().onStartup(servletContext);
        }
        catch (ServletException ex) {
            throw new ApplicationContextException("Cannot initialize servlet context",
                                                  ex);
        }
    }
    initPropertySources();
}

// #2-1-1-1  TomcatServletWebServerFactory#getwebServer
@Override
public WebServer getWebServer(ServletContextInitializer... initializers) {
    // 创建tomcat实例。 
    Tomcat tomcat = new Tomcat();
    // 创建一个临时目录，并且在jvm退出的时候删掉，调用的是tempDir.deleteOnExit()方法。
    File baseDir = (this.baseDirectory != null) ? this.baseDirectory
        : createTempDir("tomcat");
    // 设置tomcat对象的basedir属性为临时目录。
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    // 创建连接器对象，反射创建this.protocol默认的Http11NioProtocol对象，连接器的作用是处理外部请求。
    Connector connector = new Connector(this.protocol);
    // 首先实例化server=new StandardServer()，设置catalina.home和catalina.base的值，然后实例化service = new StandardService()，将server对象添加到service中，server中创建一个Service的数组，将service对象放入数组中，tomcat.getService()是获取数组中的第一个service，然后添加到connector对象中，service中创建一个Connector数组，将connector对象放入数组中。
    tomcat.getService().addConnector(connector);
    // connector定制化设置，比如设置端口号、最大线程数等参数。
    customizeConnector(connector);
    // 如果connector不在service中，那么就设置一下。
    tomcat.setConnector(connector);
    // 首先创建Engine和Host对象，然后设置host的autoDeploy自动部署为false。
    tomcat.getHost().setAutoDeploy(false);
    // 设置engine的backgroundProcessorDelay属性，如果engineValves不为空，那么engine添加所有的valves。
    configureEngine(tomcat.getEngine());
    // 如果additionalTomcatConnectors不为空，service添加所有的附加连接器。
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    // 创建TomcatEmbeddedContext上下文对象，并且添加到host的children集合中。
    prepareContext(tomcat.getHost(), initializers);
    // 启动tomcat所有的服务容器，从上到下是server->service->engine->connector，先是挨个调用init方法，然后在挨个调用start方法，这里有个地方是在engine中调用host方法的时候是用的线程池InlineExecutorService调用的host的start方法。同理，host中保存的StandardEngine[Tomcat].StandardHost[localhost].TomcatEmbeddedContext[]也是通过这种方式启动的。设置ACWSAC中的变量servletContext=applicationContextFacade对象。
    return getTomcatWebServer(tomcat);
}

# 3
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    // 如果已经注册了conversionService，那么就创建。
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    // 目的不明。
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    // 实现了LoadTimeWeaverAware接口的类首先创建。
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    // 停止使用临时的类加载器。
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    // this.configurationFrozen = true;this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);设置属性。
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    // #3-1 创建所有bean。
    beanFactory.preInstantiateSingletons();
}

// #3-1 DefaultListableBeanFactory类
@Override
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    // 创建list存储所有的beanDefinitionNames。
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    // 遍历创建所有的非懒加载的单例类。
    for (String beanName : beanNames) {
        // 根据beanName获取bean的RootBeanDefinition。
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 判断bean是非抽象的，单例，非懒加载。
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 判断是不是FactoryBean
            if (isFactoryBean(beanName)) {
                // 创建对应的bean对象并且执行各种初始化逻辑。
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                                                    ((SmartFactoryBean<?>) factory)::isEagerInit,
                                                                    getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                       ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                // 创建对应的bean对象并且执行各种初始化逻辑。
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        // 获取对应的bean。
        Object singletonInstance = getSingleton(beanName);
        // 如果实例是SmartInitializingSingleton的实现类。
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                // 执行创建完后的逻辑。
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}



```



