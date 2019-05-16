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
    //创建META-INF/spring.factories中类型为ApplicationContextInitializer的所有实例，添加到
    //initializers集合，此处getSpringFactoriesInstances为核心方法，如果想自己添加需要2步，1：创
    //建自定义类并且实现ApplicationContextInitializer接口；2：创建spring.factories文件并且配置好
    //刚刚自己定义的Initializer类，可以设置Order值来调整加载顺序。
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
    //创建META-INF/spring.factories中类型为SpringApplicationRunListener的所有实例（其实只有一个
    //EventPublishingRunListener），此实例将来可以用作驱动所有listener监听的事件，如下所示。
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //第一处listener的调用，执行listeners中唯一的一个EventPublishingRunListener实例的starting方
    //法，然后调用成员变量SimpleApplicationEventMulticaster（initialMulticaster）的
    //multicastEvent方法，其实就是调用类型为ApplicationListener监听器的onApplicationEvent方法。
    //这里有个疑问是到底调用了具体哪些listener（10个中筛选了4个）？，那么是怎么筛选出来的？？？？？
    listeners.starting();
    try {
        //创建ApplicationArguments实例
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                args);
        //#1 创建StandardServletEnvironment实例，并且加载配置资源
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                applicationArguments);
        //设置spring.beaninfo.ignore的值，先从系统参数中获取，如果没有，再从spring配置中获取。具体
        //作用不清楚。
        configureIgnoreBeanInfo(environment);
        //初始化banner标语，也就是spring加载的图画，可以创建banner.txt文件来实现自定义图案。
        Banner printedBanner = printBanner(environment);
        //根据webApplicationType类型创建对应实例，如果是Servlet,那么实例为
        //AnnotationConfigServletWebServerApplicationContext,并且初始化各种父类的实例，如：
        //GenericApplicationContext中this.beanFactory = new DefaultListableBeanFactory()；
        //关键类RootBeanDefinition也是在这个过程中注册的，并且会注册一下内置的processor。
        //AnnotatedBeanDefinitionReader对象，
        context = createApplicationContext();
        //创建FailureAnalyzers实例
        exceptionReporters = getSpringFactoriesInstances(
                SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        //上下文准备工作
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
    //第二处listener的调用，调用EventPublishingRunListener中的environmentPrepared方法，只是订阅
    //的event事件是ApplicationEnvironmentPreparedEvent。
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



