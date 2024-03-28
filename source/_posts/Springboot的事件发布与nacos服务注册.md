---
title: Springboot的事件发布与nacos服务注册
date: 2024-03-28 20:50:21
tags:
- Java
-  SpringCloud
categories: 
- Springcloud
---

## Springboot事件发布

Spring在启动的过程中会发布各类事件。比如在准备环境阶段，准备ApplicationContext等。它发布各类事件，事件订阅方能在各个阶段处理相应的业务。如在准备环境阶段，会有事件订阅放用来读取配置文件等。

## Springboot 事件处理简介

Spring的事件发布者会持有相应的Listeners，在事件发布后，根据事件类型获取对应的Linsteners，事件发布者通过invokeLinstener回调事件监听方来对事件进行处理。

### Spring启动过程事件处理

以org.springframework.boot.SpringApplication#run(java.lang.String...)方法为例来说明：

```text
ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
        configureHeadlessProperty();
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting();
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
            configureIgnoreBeanInfo(environment);
            Banner printedBanner = printBanner(environment);
            context = createApplicationContext();
            exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                    new Class[] { ConfigurableApplicationContext.class }, context);
            prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            refreshContext(context);
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
            }
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
```

总体处理流程：

SpringApplication.run() --> SpringApplicationRunListeners.starting()-->EventPublishingRunListener.starting()-->SimpleApplicationEventMulticaster.multicastEvent()-->SimpleApplicationEventMulticaster.invokeListener()-->SimpleApplicationEventMulticaster.doInvokeListener()-->

最后在doInvokeListener总，来回调事件订阅方：

```text
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
        try {
            listener.onApplicationEvent(event);
        }
        catch (ClassCastException ex) {
            String msg = ex.getMessage();
            if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
                // Possibly a lambda-defined listener which we could not resolve the generic event type for
                // -> let's suppress the exception and just log a debug message.
                Log logger = LogFactory.getLog(getClass());
                if (logger.isTraceEnabled()) {
                    logger.trace("Non-matching event type for listener: " + listener, ex);
                }
            }
            else {
                throw ex;
            }
        }
    }
```

其他说明：

- SpringApplicationRunListeners持有SpringApplicationRunListener接口对象EventPublishingRunListener  

- EventPublishingRunListener 持有SimpleApplicationEventMulticaster，在EventPublishingRunListener初始化时会给相应SimpleApplicationEventMulticaster对象赋值  

### step1 获取listeners

通过getRunListeners(args)获取到SpringApplicationRunListeners对象。SpringApplicationRunListeners是对SpringApplicationRunListener的进一步封装，它持有一个List。它的实现如下：

```text
private SpringApplicationRunListeners getRunListeners(String[] args) {
        Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
        return new SpringApplicationRunListeners(logger,
                getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
    }
```

从getSpringFactoriesInstances方法便知：它应该是从Spring.factories文件中读取的（别问我是怎么知道的）。果然，在Spring.factories中可以看到：

```text
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

口说无凭，来看下面这图：

![](https://pic2.zhimg.com/80/v2-783196f7e6498b4f4b90910c02e336d1_720w.webp)

EventPublishingRunListener

它的调用栈是在SpringApplicationRunListeners调用starting开始的。

此后，EventPublishingRunListen便开始了事件发布：

```text
public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();
        for (ApplicationListener<?> listener : application.getListeners()) {
            this.initialMulticaster.addApplicationListener(listener);
        }
    }

    @Override
    public int getOrder() {
        return 0;
    }

public void starting() {
        this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
    }
```

由上可知：

- 它持有事件发布者initialMulticaster。它是一个SimpleApplicationEventMulticaster对象  

- SimpleApplicationEventMulticaster是一个事件发布方，在EventPublishingRunListener的构造函数中，会初始化它发布的事件相关的listener。  

- 通过事件发布者，发布了一个ApplicationStartingEvent事件。  

### step2 事件处理

事件处理以SimpleApplicationEventMulticaster处理事件为例。

```text
@Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        Executor executor = getTaskExecutor();
        for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            }
            else {
                invokeListener(listener, event);
            }
        }
    }
```

在发布事件时，会调用通过getApplicationListeners方法，根据实际事件event获取listener。通俗来说，就是获取到关注发布的事件的对象，然后通过invokeListener来进行回调

## nacos 服务注册

### Springcloud 服务注册原理

在介绍nacos服务注册前，先介绍下SpringCloud服务注册的原理：

SpringCloud服务注册，是基于Springboot的事件机制。在spring-cloud-common包的AbstractAutoServiceRegistration中，订阅了WebServerInitializedEvent事件，这样，Springcloud应用启动的时候的某一阶段会发布WebServerInitializedEvent事件。按照之前介绍的事件处理机制，AbstractAutoServiceRegistration的实现类可以接收到这个事件，继而实现服务注册。

AbstractAutoServiceRegistration在接收到WebServerInitializedEvent事件后的处理流程如下：

AbstractAutoServiceRegistration.onApplicationEvent--> AbstractAutoServiceRegistration.bind-->AbstractAutoServiceRegistration.start()-->AbstractAutoServiceRegistration.register()-->ServiceRegistry.register(getRegistration())实现服务注册。

在nacos 中，同样通过Spring-cloud定义的标准，实现了服务注册:

- NacosAutoServiceRegistration实现了AbstractAutoServiceRegistration  

- NacosServiceRegistry实现了ServiceRegistry接口  

- nacos在收到WebServerInitializedEvent事件后，调用NacosServiceRegistry实现了服务注册。  

根据Springboot的事件以及事件监听者的回调，基本上理清了nacos服务注册时的大致流程。但是还存在一个问题：WebServerInitializedEvent事件是什么时候，由谁发布的？

### Lifecycle与WebServerInitializedEvent事件发布与

众所周知：Spring ApplicationContext的核心方法refresh，会实例化所有singleton对象。来看看refresh中的最后一个方法finishRefresh：

```text
protected void finishRefresh() {
        // Clear context-level resource caches (such as ASM metadata from scanning).
        clearResourceCaches();

        // Initialize lifecycle processor for this context.
        initLifecycleProcessor();

        // Propagate refresh to lifecycle processor first.
        getLifecycleProcessor().onRefresh();

        // Publish the final event.
        publishEvent(new ContextRefreshedEvent(this));

        // Participate in LiveBeansView MBean, if active.
        LiveBeansView.registerApplicationContext(this);
    }
```

实际上，WebServerInitializedEvent也是在finishRefresh中完成发布的。

### WebServerInitializedEvent的发布过程

AbstractApplicationContext.finishRefresh()-->DefaultLifecycleProcessor.onRefresh()-->

DefaultLifecycleProcessor.startBeans()-->DefaultLifecycleProcessor.getLifecycleBeans()-->DefaultLifecycleProcessor.doStart()-->WebServerStartStopLifecycle.start-()

上面有一些DefaultLifecycleProcessor的内部类调用链路省略了，有兴趣可以看看：

![](https://pic1.zhimg.com/80/v2-23a40f3f3aa4f5513b4db3ca98b38e28_720w.webp)

lifecycle调用链

总体来说，就是DefaultLifecycleProcessor会调用实现了Lifecycle接口的start方法，而WebServerStartStopLifecycle也实现了Lifecycle，因此会在此时被调用。

而WebServerStartStopLifecycle在它的start 方法中，会发布ServletWebServerInitializedEvent事件。

```text
WebServerStartStopLifecycle(ServletWebServerApplicationContext applicationContext, WebServer webServer) {
        this.applicationContext = applicationContext;
        this.webServer = webServer;
    }

    @Override
    public void start() {
        this.webServer.start();
        this.running = true;
        this.applicationContext
                .publishEvent(new ServletWebServerInitializedEvent(this.webServer, this.applicationContext));
    }
```

## 总结

- 在Springcloud中，在程序启动过程中，使用的是Springboot中的ServletWebServerApplicationContext作为容器。在它实例化singleton对象后最后阶段会在运行finishRefresh方法时发布ServletWebServerInitializedEvent事件。
- 在nacos中，通过NacosAutoServiceRegistration监听到WebServerInitializedEvent事件（WebServerInitializedEvent继承自ServletWebServerInitializedEvent），然后调用NacosRegistration实现了服务的自动注册。
