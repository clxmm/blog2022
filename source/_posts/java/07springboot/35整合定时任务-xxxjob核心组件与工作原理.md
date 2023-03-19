---
title: 35整合定时任务-xxxjob核心组件与工作原理
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

简单使用了 xxl-job 之后，从装配的内容上也能看得出来，只需要向 IOC 容器中注册一个 `XxlJobSpringExecutor` ，就能完成与 `xxl-job-admin` 的通信，这个组件是如何发挥作用的？`xxl-job-admin` 又是如何将定时任务派发给各个业务服务的呢？本章我们就来分析其中的原理。

在这之前，小册先贴出 xxl-job 的官方文档中的一张架构图，帮助各位从整体上了解 xxl-job 的架构。


<!--more-->

![](../img/202303/35xxx-job1.png)

## 1. XxlJobSpringExecutor

我们先来看一下 `XxlJobSpringExecutor` 的继承关系和实现的接口，这个类的类名上有 Spring 的字样，那它肯定是跟 SpringFramework 有关联的。翻开它的源码，可以发现它采用两个接口将自身的生命周期与 IOC 容器的生命周期相挂钩：

```java
public class XxlJobSpringExecutor extends XxlJobExecutor 
  	implements ApplicationContextAware, SmartInitializingSingleton, DisposableBean

```

除此之外，它继承的 xxl-job 原生不依赖任何框架的 `XxlJobExecutor` 的内部则是定义了对象创建时需要注入的那些属性，另外还有它的生命周期相关方法定义。

```java
public class XxlJobExecutor  {
    private String adminAddresses;
    private String accessToken;
    private String appname;
    private String address;
    private String ip;
    private int port;
    private String logPath;
    private int logRetentionDays;
    // ......
```

说白了，`XxlJobSpringExecutor` 能正常工作，就要从它的生命周期入手观察，下面我们就来一一深入。

### 1.1 afterSingletonsInstantiated

`XxlJobSpringExecutor` 初始化的时机选用的 `SmartInitializingSingleton` 作为切入点，对应的 `afterSingletonsInstantiated` 中包含两个核心步骤值得我们深入研究：

```java
    @Override
    public void afterSingletonsInstantiated() {
        // 1.1.1 初始化Bean方法的任务调度器
        initJobHandlerMethodRepository(applicationContext);
        // 该步骤为xxl-job提供的Glue模式，小册不展开讲解了
        GlueFactory.refreshInstance(1);
        try {
            // 1.1.2 回调生命周期启动
            super.start();
        } // catch throw ex ......
    }
```

#### 1.1.1 initJobHandlerMethodRepository

为什么选用 `SmartInitializingSingleton` 作为切入的初始化时机，而不是 `InitializingBean` 呢？那是因为在 `InitializingBean` 的 `afterPropertiesSet` 方法回调时，IOC 容器中的非延迟加载的单实例 Bean 尚未初始化完毕，所以那个时候拿到的 Bean 不是最全的；而 `SmartInitializingSingleton` 的 `afterSingletonsInstantiated` 方法被回调时，IOC 容器中的非延迟加载的单实例 Bean 已经全部初始化完毕，这个时候拿到的 Bean 是最全的。

##### 1.1.1.1 方法实现

下面我们看一下这个方法的实现，从源码的逻辑上看，它做的事情其实就是把 IOC 容器中的 Bean 全部取出，并逐个扫描内部的方法有没有标注 `@XxlJob` 注解（可以说是非常简单粗暴了），如果有取出标注的方法，那就把这些方法封装为一个一个的 `MethodJobHandler` 。

```java
private void initJobHandlerMethodRepository(ApplicationContext applicationContext) {
    if (applicationContext == null) {
        return;
    }
    // init job handler from method
    String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
    for (String beanDefinitionName : beanDefinitionNames) {
        Object bean = applicationContext.getBean(beanDefinitionName);

        Map<Method, XxlJob> annotatedMethods = null;
        try {
            annotatedMethods = MethodIntrospector.selectMethods(bean.getClass(),
                    new MethodIntrospector.MetadataLookup<XxlJob>() {
                        @Override
                        public XxlJob inspect(Method method) {
                            return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
                        }
                    });
        } // catch logger ......
        if (annotatedMethods==null || annotatedMethods.isEmpty()) {
            continue;
        }

        for (Map.Entry<Method, XxlJob> methodXxlJobEntry : annotatedMethods.entrySet()) {
            Method executeMethod = methodXxlJobEntry.getKey();
            XxlJob xxlJob = methodXxlJobEntry.getValue();
            registJobHandler(xxlJob, bean, executeMethod);
        }
    }
}
```

毫无疑问，这段源码中最值得深入的就是最后的那句 `registJobHandler` 方法了，我们深入进去。

##### 1.1.1.2 registJobHandler

```java
protected void registJobHandler(XxlJob xxlJob, Object bean, Method executeMethod){
    // 检查 ......
    String name = xxlJob.value();
    Class<?> clazz = bean.getClass();
    String methodName = executeMethod.getName();
    // 检查 ......

    // 兼容protected等方法
    executeMethod.setAccessible(true);

    // 如果标注的@XxlJob注解上还声明了init和destroy方法，则此处会一并记录下来
    Method initMethod = null;
    Method destroyMethod = null;
    if (xxlJob.init().trim().length() > 0) {
        try {
            initMethod = clazz.getDeclaredMethod(xxlJob.init());
            initMethod.setAccessible(true);
        } // catch throw ex ......
    }
    if (xxlJob.destroy().trim().length() > 0) {
        try {
            destroyMethod = clazz.getDeclaredMethod(xxlJob.destroy());
            destroyMethod.setAccessible(true);
        } // catch throw ex ......
    }

    // registry jobhandler
    registJobHandler(name, new MethodJobHandler(bean, executeMethod, initMethod, destroyMethod));
}
```

从上面的源码中我们可以看出，最核心的代码还是最底下的那句 `registJobHandler` 方法，不过在这之前我们可以留意两个细节：

- 1）`executeMethod.setAccessible(true);` 这句代码使得 xxl-job 准许我们把 `@XxlJob` 注解标注在 `protected` 甚至 `private` 方法上！
- 2）`@XxlJob` 注解中还有两个属性：`init` 和 `destroy` ，这两个方法可以分别引用当前标注的方法所在类的其他两个方法，在定时任务方法被执行的前后，会分别执行这两个方法。

##### 1.1.1.3 重载的registJobHandler

下面我们继续深入这个重载的 `registJobHandler` 方法，走到这里可以发现，其实 `registJobHandler` 方法就是把刚创建好的 `MethodJobHandler` 对象放入一个 `Map` 中，并没有什么很神秘的操作了。

```java
private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = 
  new ConcurrentHashMap<String, IJobHandler>();

public static IJobHandler registJobHandler(String name, IJobHandler jobHandler) {
    logger.info(">>>>>>>>>>> xxl-job register jobhandler success, name:{}, jobHandler:{}", name, jobHandler);
    return jobHandlerRepository.put(name, jobHandler);
}
```

简单总结一下，`initJobHandlerMethodRepository` 方法完成的工作，就是将我们在业务代码中标注了 `@XxlJob` 注解的方法，封装为一个一个的 `MethodJobHandler` ，顺便还给它们扩展了触发前后的生命周期逻辑。

#### 1.1.2 super.start

初始化完所有的 `MethodJobHandler` 后，下一步便是回调自身的 `start` 方法初始化底层了。来到父类 `XxlJobExecutor` 中，可以发现初始化包含 5 个步骤：

```java
public void start() throws Exception {
    // init logpath 初始化存放执行日志目录文件
    XxlJobFileAppender.initLogPath(logPath);
    // init invoker, admin-client 初始化执行者，管理客户端
    initAdminBizList(adminAddresses, accessToken);

    // init JobLogFileCleanThread 初始化日志清除线程
    JobLogFileCleanThread.getInstance().start(logRetentionDays);
    // init TriggerCallbackThread 初始化回调触发器线程
    TriggerCallbackThread.getInstance().start();
    // init executor-server 初始化执行服务器
    initEmbedServer(address, ip, port, appname, accessToken);
}
```

这里面我们重点关心的是与任务调度和执行有关的第 2 、4 、5 步，与日志相关的内容我们就不多关心了。

##### 1.1.2.1 initAdminBizList - 初始化执行者，管理客户端

```java

private static List<AdminBiz> adminBizList;

private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
    if (adminAddresses!=null && adminAddresses.trim().length()>0) {
        for (String address: adminAddresses.trim().split(",")) {
            if (address!=null && address.trim().length()>0) {
                AdminBiz adminBiz = new AdminBizClient(address.trim(), accessToken);
                if (adminBizList == null) {
                    adminBizList = new ArrayList<AdminBiz>();
                }
                adminBizList.add(adminBiz);
            }
        }
    }
}
```

前面我们说了，集成了 `xxl-job` 的业务服务跟 `xxl-job-admin` 有一个简单的服务注册与发现机制，当被调度的业务服务启动时，应该将自身的信息注册到 `xxl-job-admin` 中。`initAdminBizList` 方法的内部就会将这些配置好的 `xxl-job-admin` 的客户端地址，逐个封装为 `AdminBizClient` 对象。

这个 `AdminBizClient` 是怎么做到服务注册的呢？我们可以点进去看一眼：

```java
public ReturnT<String> registry(RegistryParam registryParam) {
    return XxlJobRemotingUtil.postBody(addressUrl + "api/registry", accessToken, timeout, registryParam, String.class);
}
```

好家伙，这里面定义了一个 `registry` 方法，这不就是向 `xxl-job-admin` 注册自身的信息嘛。所以这么看下来，这个 `initAdminBizList` 环节执行完毕后，业务服务就可以向 `xxl-job-admin` 注册自身的信息了。

##### 1.1.2.2 TriggerCallbackThread - 初始化回调触发器线程

下一个关键的步骤是初始化回调的触发器线程，这是个什么东西呢？我们点进去 `TriggerCallbackThread` 中看一下这里面都藏了什么。

```java
private Thread triggerCallbackThread;
private Thread triggerRetryCallbackThread;
private volatile boolean toStop = false;

public void start() {
    // 校验 ......

    // callback 回调的线程
    triggerCallbackThread = new Thread(new Runnable() {
        // ......
    });
    triggerCallbackThread.setDaemon(true);
    triggerCallbackThread.setName("xxl-job, executor TriggerCallbackThread");
    triggerCallbackThread.start();

    // retry 重试的线程
    triggerRetryCallbackThread = new Thread(new Runnable() {
        // ......
    });
    triggerRetryCallbackThread.setDaemon(true);
    triggerRetryCallbackThread.start();
}
```

很明显，`TriggerCallbackThread` 中包含了两个 Thread 线程，它们负责的内容分别是：

- `triggerCallbackThread` ：定时任务被调用后，回调告诉 `xxl-job-admin` 这个任务的执行结果，由 `xxl-job-admin` 将调度结果写入数据库；
- `triggerRetryCallbackThread` ：如果定时任务调用失败，则这个线程会告诉 `xxl-job-admin` ，让调度中心去重试任务调度；

很明显，只有这两个线程正确工作，才能让调度中心跟业务服务之间的配合达到我们预想的效果，所以这两个线程也应该在工程启动时初始化好。

至于线程的内部都做了什么，各位可以参照 IDE 去翻看源码，这部分的实现逻辑有点长，小册就不再贴出了。

##### 1.1.2.3 initEmbedServer - 初始化执行服务器

调度中心监测到定时任务要执行的时候，自然需要通知某一个业务服务，触发它的方法才可以，那我们编写的那些业务服务如何才能感知到调度中心的通知呢？很明显，它肯定也是通过暴露一些接口 / 端点来实现的。xxl-job 的内部集成了一个 Netty ，在工程启动的时候，这个 `initEmbedServer` 方法会在底层启动一个 Netty 服务器，由它来接收来自调度中心的调度通知。

```java
private void initEmbedServer(String address, String ip, int port, String appname, 
        String accessToken) throws Exception {
    // fill ip port
    port = port > 0 ? port : NetUtil.findAvailablePort(9999);
    ip = (ip != null && ip.trim().length() > 0) ? ip : IpUtil.getIp();

    // generate address
    if (address == null || address.trim().length() == 0) {
        String ip_port_address = IpUtil.getIpPort(ip, port);
        address = "http://{ip_port}/".replace("{ip_port}", ip_port_address);
    }

    // accessToken
    if (accessToken == null || accessToken.trim().length() == 0) {
        logger.warn(">>>>>>>>>>> xxl-job accessToken is empty. To ensure system security, please set the accessToken.");
    }

    // start
    embedServer = new EmbedServer();
    embedServer.start(address, port, appname, accessToken);
}
```

看上面这段源码的最后两行，这就是初始化 Netty 的位置，在最后执行的 `start` 方法内部，它就会另起一个线程，手动构造一个 `ServerBootstrap` ，随后在一些逻辑后绑定产生一个 `ChannelFuture` ，这其实就是 Netty 的体现了。如果各位对 Netty 不甚了解，其实也没关系，大不了你把它当作一个 Tomcat 来看嘛。。。我们只需要知道的是，这个内部的 Netty 服务器启动之后，就可以接收 `xxl-job-admin` 的任务调度了。

```java
private ExecutorBiz executorBiz;
private Thread thread;

public void start(final String address, final int port, final String appname, final String accessToken) {
    executorBiz = new ExecutorBizImpl();
    thread = new Thread(new Runnable() {
        @Override
        public void run() {
            // param
            EventLoopGroup bossGroup = new NioEventLoopGroup();
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            ThreadPoolExecutor bizThreadPool = new ThreadPoolExecutor(...); // 初始化线程池
            try {
                // start server
                ServerBootstrap bootstrap = new ServerBootstrap();
                bootstrap.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class)
                        .childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            public void initChannel(SocketChannel channel) throws Exception {
                                channel.pipeline()
                                        .addLast(new IdleStateHandler(0, 0, 30 * 3, TimeUnit.SECONDS))  // beat 3N, close if idle
                                        .addLast(new HttpServerCodec())
                                        .addLast(new HttpObjectAggregator(5 * 1024 * 1024))  // merge request & reponse to FULL
                                        .addLast(new EmbedHttpServerHandler(executorBiz, accessToken, bizThreadPool));
                            }
                        })
                        .childOption(ChannelOption.SO_KEEPALIVE, true);
                // 一些省略的源码 ......
                ChannelFuture future = bootstrap.bind(port).sync();
                // 一些省略的源码 ......
            } // catch finally ......
        }
    });
    thread.setDaemon(true);    // daemon, service jvm, user thread leave >>> daemon leave >>> jvm leave
    thread.start();
}
```

注意上面源码的一个细节：在初始化 Netty 的 `SocketChannel` 时，它向最后添加了一个 `EmbedHttpServerHandler` ，这个组件在后面讲解任务调度全流程分析时会涉及到，各位先混个脸熟即可。

经过上述的源码逻辑后，`XxlJobSpringExecutor` 就已经初始化并启动了，也就可以接收来自 `xxl-job-admin` 的任务调度了。

### 1.2 destroy

看完了初始化，我们再来看一下销毁。相比较起来，销毁的逻辑就简单多了，从 `XxlJobSpringExecutor` 的 `destroy` 方法上也能看的出来，它并没有扩展更多逻辑，只是简单地调用了父类的 `destroy` 方法。

```java
@Override
public void destroy() {
    super.destroy();
}
```

而来到父类 `XxlJobExecutor` 的 destroy 方法中可以看到，整个过程其实就是上面初始化的逆序销毁，我们可以简单读一下这段源码。

1）先关闭内置的 Netty 服务器，毕竟先斩断与外界的联系，可以保证不会再有新的定时任务被触发；

2）销毁 Thread 和 Handler ，这样即便再触发任务，也找不到对应执行的方法了；

3）销毁与日志相关的线程组件，这样也不会有日志打印了。

```java
public void destroy() {
    // destroy executor-server
    stopEmbedServer();

    // destroy jobThreadRepository
    if (jobThreadRepository.size() > 0) {
        for (Map.Entry<Integer, JobThread> item: jobThreadRepository.entrySet()) {
            JobThread oldJobThread = removeJobThread(item.getKey(), "web container destroy and kill the job.");
            // wait for job thread push result to callback queue
            if (oldJobThread != null) {
                try {
                    oldJobThread.join();
                } // catch logger ......
            }
        }
        jobThreadRepository.clear();
    }
    jobHandlerRepository.clear();

    // destroy JobLogFileCleanThread
    JobLogFileCleanThread.getInstance().toStop();
    // destroy TriggerCallbackThread
    TriggerCallbackThread.getInstance().toStop();
}
```

## 2. 一个任务调度的全过程分析

---

看完 `XxlJobSpringExecutor` 的初始化和销毁机制后，下面我们以前一章中的示例，从源码的执行过程感受 xxl-job 的调度全流程。

### 2.1 执行触发动作

由于定时任务的触发是在 `xxl-job-admin` 中定时触发，这个逻辑的上手门槛略高，小册选择使用控制台中“执行一次”的逻辑作为切入点来分析。

在任务调度控制台上触发 `demoTest` 任务后，它给 `xxl-job-admin` 发送的请求是 `/jobinfo/trigger` ，相应地触发后台的 Controller 方法为 `com.xxl.job.admin.controller.JobInfoController` ，源码如下。

```java
@RequestMapping("/trigger")
@ResponseBody
public ReturnT<String> triggerJob(int id, String executorParam, String addressList) {
    // force cover job param
    if (executorParam == null) {
        executorParam = "";
    }

    JobTriggerPoolHelper.trigger(id, TriggerTypeEnum.MANUAL, -1, null, executorParam, addressList);
    return ReturnT.SUCCESS;
}
```

可以发现，触发一次任务调度的方法会使用一个帮助类 `JobTriggerPoolHelper` 来转发执行，注意 `trigger` 传入的参数中有一个 `TriggerTypeEnum.MANUAL` ，它代表的就是单次方法执行，如果是使用 cron 表达式模式执行的话，那传入的则是 `TriggerTypeEnum.CRON` 。

顺着 `trigger` 方法往下走，可以发现它执行的是 `helper` 的 `addTrigger` 方法，而再往下深入，会发现它使用自身集成的线程池，提交了一个任务调度的线程过去。

```java
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, 
        String executorShardingParam, String executorParam, String addressList) {
    helper.addTrigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam, addressList);
}

public void addTrigger(final int jobId, final TriggerTypeEnum triggerType, final int failRetryCount,
        final String executorShardingParam, final String executorParam, final String addressList) {
    // choose thread pool
    ThreadPoolExecutor triggerPool_ = fastTriggerPool;
    AtomicInteger jobTimeoutCount = jobTimeoutCountMap.get(jobId);
    // job-timeout 10 times in 1 min
    if (jobTimeoutCount!=null && jobTimeoutCount.get() > 10) {
        triggerPool_ = slowTriggerPool;
    }

    // trigger
    triggerPool_.execute(new Runnable() {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            try {
                // do trigger
                XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam, addressList);
            } // catch finally ......
        }
    });
}
```

最底下的方法就是提交执行线程的方法了，这里面的核心是 `XxlJobTrigger.trigger` 方法，下面的那些 catch 和 finally 就不关心了。

### 2.2 XxlJobTrigger.trigger

下面这段源码比较长，大家可以结合着源码中标注的注释来看。

```java
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount,
        String executorShardingParam, String executorParam, String addressList) {
    // load data 从数据库中查出这个任务调度的信息
    XxlJobInfo jobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(jobId);
    // 判空检查 ......
    if (executorParam != null) {
        jobInfo.setExecutorParam(executorParam);
    }
    int finalFailRetryCount = failRetryCount>=0?failRetryCount:jobInfo.getExecutorFailRetryCount();
    XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());

    // cover addressList
    if (addressList!=null && addressList.trim().length()>0) {
        group.setAddressType(1);
        group.setAddressList(addressList.trim());
    }

    // sharding param 如果需要分片的话，要考虑任务调度的分片策略
    int[] shardingParam = null;
    if (executorShardingParam!=null){
        String[] shardingArr = executorShardingParam.split("/");
        if (shardingArr.length==2 && isNumeric(shardingArr[0]) && isNumeric(shardingArr[1])) {
            shardingParam = new int[2];
            shardingParam[0] = Integer.valueOf(shardingArr[0]);
            shardingParam[1] = Integer.valueOf(shardingArr[1]);
        }
    }
    // 广播模式下，需要对所有指定的服务实例都触发一次
    if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
            && group.getRegistryList()!=null && !group.getRegistryList().isEmpty()
            && shardingParam==null) {
        for (int i = 0; i < group.getRegistryList().size(); i++) {
            processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());
        }
    } else {
        // 普通的触发，只调用一个业务服务的实例
        if (shardingParam == null) {
            shardingParam = new int[]{0, 1};
        }
        processTrigger(group, jobInfo, finalFailRetryCount, triggerType, shardingParam[0], shardingParam[1]);
    }
}
```

简单概括一下这段源码的逻辑，其实很简单，就是 `xxl-job-admin` 负责将这个要触发的定时任务的定义信息，从数据库中查出来，并且根据这个任务的分发情况，决定是广播调度还是单体调度。但无论怎么调度，最终执行的方法都是 `processTrigger` ，我们继续往下看。

### 2.3 processTrigger

接下来这个方法的篇幅更长，为了让各位能准确捕捉方法执行的核心逻辑，小册把一些与本次我们研究的主干流程关系不大的源码都省略掉了，各位如果想看全部的源码，可以借助 IDE 来查看。

```java
private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, 
        TriggerTypeEnum triggerType, int index, int total) {
    // param
    ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(jobInfo.getExecutorBlockStrategy(), 
             ExecutorBlockStrategyEnum.SERIAL_EXECUTION);  // block strategy
    ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);
    String shardingParam = (ExecutorRouteStrategyEnum.SHARDING_BROADCAST == executorRouteStrategyEnum) 
            ? String.valueOf(index).concat("/").concat(String.valueOf(total)) : null;

    // 1、save log-id 省略 ......

    // 2、init trigger-param
    TriggerParam triggerParam = new TriggerParam();
    triggerParam.setJobId(jobInfo.getId());
    triggerParam.setExecutorHandler(jobInfo.getExecutorHandler());
    triggerParam.setExecutorParams(jobInfo.getExecutorParam());
    // 还有好多setter ......

    // 3、init address 省略 ......

    // 4、trigger remote executor 执行远程的任务执行器
    ReturnT<String> triggerResult = null;
    if (address != null) {
        triggerResult = runExecutor(triggerParam, address);
    } else {
        triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
    }

    // 收到执行结果的处理 省略 ......
}
```

仔细观察保留下的源码，可以发现整个逻辑中最核心的方法就是封装 `TriggerParam` ，加载业务服务的地址后，发起远程的请求，通知业务服务执行对应的定时任务逻辑（如果没有寻找到业务服务的地址则直接返回失败）。

向下执行 `runExecutor` 方法时，它会使用 `XxlJobScheduler` 去获取到支持远程请求的客户端对象 `ExecutorBizClient` ，并由它向业务服务的 Netty 发送一个 POST 请求，告诉它要触发执行的任务执行器。

```java
public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
    ReturnT<String> runResult = null;
    try {
        ExecutorBiz executorBiz = XxlJobScheduler.getExecutorBiz(address);
        runResult = executorBiz.run(triggerParam);
    } // catch logger ......
    // ......
    return runResult;
}

public ReturnT<String> run(TriggerParam triggerParam) {
    return XxlJobRemotingUtil.postBody(addressUrl + "run", accessToken, timeout, triggerParam, String.class);
}
```

那下面要做的事情就要来到定义任务的那些业务服务端了，在本次测试中也就相当于要来到 `spring-boot-integration-09-xxljob` 中了。

### 2.4 来到业务服务端

负责接收 `xxl-job-admin` 请求的处理器是 `EmbedHttpServerHandler` ，这个在上面 1.1.2.3 小节已经提到过了。这个类定义在 `EmbedServer` 中，以内部类的形式体现。

```java
public static class EmbedHttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest>

```

`xxl-job` 跟 `xxl-job-admin` 的交互毕竟不是很多，很明显不需要大张旗鼓的用 MVC 系列的框架来制作，`xxl-job` 的开发者考虑到最小实现的原则，于是它使用了 Netty 的最简易实现：uri 匹配的方式。对应的具体实现方法是下面的 `channelRead0` 方法，这个方法也是通过提交一个执行线程的方式来实现，而线程中做的事其实就是一次处理请求的三个步骤：执行可以处理当前请求的方法、将方法的返回值序列化为 json 字符串、将 json 响应给 xxl-job-admin 。

```java
rotected void channelRead0(final ChannelHandlerContext ctx, FullHttpRequest msg) throws Exception {
    // request parse
    //final byte[] requestBytes = ByteBufUtil.getBytes(msg.content());    // byteBuf.toString(io.netty.util.CharsetUtil.UTF_8);
    String requestData = msg.content().toString(CharsetUtil.UTF_8);
    String uri = msg.uri();
    HttpMethod httpMethod = msg.method();
    boolean keepAlive = HttpUtil.isKeepAlive(msg);
    String accessTokenReq = msg.headers().get(XxlJobRemotingUtil.XXL_JOB_ACCESS_TOKEN);

    // invoke
    bizThreadPool.execute(new Runnable() {
        @Override
        public void run() {
            // do invoke 执行方法
            Object responseObj = process(httpMethod, uri, requestData, accessTokenReq);
            // to json 将响应结果序列化为json
            String responseJson = GsonTool.toJson(responseObj);
            // write response 将返回结果响应给xxl-job-admin
            writeResponse(ctx, keepAlive, responseJson);
        }
    });
}
```

那具体如何判断一个请求应该由哪个方法来执行呢？我们进入 process 方法中查看。

### 2.5 process

```java
private Object process(HttpMethod httpMethod, String uri, String requestData, String accessTokenReq) {
    // 校验 ......

    // services mapping
    try {
        switch (uri) {
            case "/beat":
                return executorBiz.beat();
            case "/idleBeat":
                IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
                return executorBiz.idleBeat(idleBeatParam);
            case "/run":
                TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
                return executorBiz.run(triggerParam);
            case "/kill":
                KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
                return executorBiz.kill(killParam);
            case "/log":
                LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
                return executorBiz.log(logParam);
            default:
                return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping(" + uri + ") not found.");
        }
    } // catch return ......
}
```

从源码中很容易能看出，在进行一些简单的前置校验后，它使用 switch 判断结构来匹配当前请求的 uri ，并根据不同的 uri 执行 `ExecutorBiz` 的不同方法。

很明显，上面执行的方法是 `run` ，我们继续向内部深入。

### 2.6 触发run方法

注意，此时来到的类是 `ExecutorBizImpl` 。这个 `run` 方法的篇幅很长啊，所以我们依然是省略一部分与我们本次研究的场景无关的源码，以保证各位能准确捕获关键的流程。

```java
@Override
public ReturnT<String> run(TriggerParam triggerParam) {
    // load old：jobHandler + jobThread
    JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
    IJobHandler jobHandler = jobThread != null ? jobThread.getHandler() : null;
    String removeOldReason = null;

    // valid：jobHandler + jobThread
    GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
    if (GlueTypeEnum.BEAN == glueTypeEnum) {
        // new jobhandler
        IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());
        // valid old jobThread handler 省略 ......
    } // else if ......

    // executor block strategy 省略 ......

    // replace thread (new or exists invalid)
    if (jobThread == null) {
        jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }

    // push data to queue
    ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
    return pushResult;
}

```

从方法的核心脉络上不难看出，整个过程包含获取 `IJobHandler` （也即上面封装的 `MethodJobHandler` 对象）、校验线程和执行器是否正确，随后就是执行这个 `IJobHandler` 了。请注意，执行 `IJobHandler` 的动作叫 `registJobThread` ，而不是叫 `executeJobHandler` ，为什么会这么命名呢？

### 2.7 registJobThread

注册，不是执行，那就有可能是先不着急执行（或不是立即执行的）。结合前面看过的那么多源码，各位是否会产生一个猜想：不会这个 `registJobThread` 也是搞线程那一套吧！显而易见，进入这个方法后可以发现的确是这样的，它会将 `IJobHandler` 封装到一个 `JobThread` 中，随后调用它的 `start` 方法（很明显是 `Thread` 的 `start` 方法了）。

```java
private static ConcurrentMap<Integer, JobThread> jobThreadRepository = new ConcurrentHashMap<>();

public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason) {
    JobThread newJobThread = new JobThread(jobId, handler);
    newJobThread.start();
    logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[] {jobId, handler});

    // putIfAbsent | oh my god, map's put method return the old value!!!
    JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread);
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();
    }
    return newJobThread;
}
```

那毫无疑问，我们接下来要进入 `JobThread` 的 `run` 方法中一探究竟了。非常令人头痛的是，这个方法的源码篇幅更长，甚至小册尝试省略大部分源码后，最终还是留下了近 50 行。不过剥离出核心的几个步骤还是很简单的，我们只需要看开头、中间、结尾即可。

```java
@Override
public void run() {
    try {
        handler.init(); // 1. 执行目标方法前的自定义初始化动作
    } // catch logger ......

    // execute
    while(!toStop){
        running = false;
        idleTimes++;

        TriggerParam triggerParam = null;
        try {
            triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
            if (triggerParam != null) {
                // 好多前置准备 ......
                XxlJobContext xxlJobContext = new XxlJobContext( triggerParam.getJobId(), triggerParam.getExecutorParams(), 
                        logFileName, triggerParam.getBroadcastIndex(), triggerParam.getBroadcastTotal()); 
                // init job context
                XxlJobContext.setXxlJobContext(xxlJobContext);
                // execute logger ......

                if (triggerParam.getExecutorTimeout() > 0) {
                    // 异步执行 省略 ......
                } else {
                    // just execute 2. 同步执行，此处就是真正的方法执行
                    handler.execute();
                }
                // valid execute handle data 校验逻辑 省略 ......
            } else {
                if (idleTimes > 30) {
                    if(triggerQueue.size() == 0) {  // avoid concurrent trigger causes jobId-lost
                        XxlJobExecutor.removeJobThread(jobId, "excutor idel times over limit.");
                    }
                }
            }
        } // catch finally ......
    }

    // callback trigger request in queue 向xxl-job-admin响应调度成功的结果 省略 ......

    try {
        handler.destroy(); // 3. 执行目标方法后的自定义销毁动作
    } // catch logger ......
    // logger ......
}
```

1. `handler.init();` 反射执行目标方法前的自定义初始化动作

   ```java
   public void init() throws Exception {
       if (initMethod != null) {
           initMethod.invoke(target);
       }
   }
   ```

2.`handler.execute();` 反射执行被 `@XxlJob` 注解标注的目标方法

```java
public void execute() throws Exception {
    Class<?>[] paramTypes = method.getParameterTypes();
    if (paramTypes.length > 0) {
        // method-param can not be primitive-types 带参数的反射调用
        method.invoke(target, new Object[paramTypes.length]);
    } else {
        // 没有任何参数的反射调用
        method.invoke(target);
    }
}
```

3.`handler.destroy();` 反射执行目标方法后的自定义销毁动作

```java
public void destroy() throws Exception {
    if (destroyMethod != null) {
        destroyMethod.invoke(target);
    }
}
```

这样一抽取出来，整个 `run` 方法的过程就很明晰了吧，其实就是三个反射动作。

到此为止，从 `xxl-job-admin` 开始的任务调度，到业务服务的调度完成，整个过程的核心动作也就都看完了，至于调度完成后 xxl-job 如何告诉 xxl-job-admin ，相对来讲逻辑就简单一些了，小册就不再展开讲解了，感兴趣的小伙伴可以自行查看。

【好啦，与定时任务调度相关的 3 章内容也就结束了。xxl-job 在微服务项目开发中使用的还是很多的，各位如果需要使用其中的特性，完全可以参照官方文档来使用，小册做的也是帮各位梳理其中的核心组件、初始化销毁流程，以及工作时的核心流程】