---
title: 33整合定时任务-Quartz实现定时任务
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

前面的 4 章中，我们完成了对 RocketMQ 的整合、使用和原理探究，接下来我们来玩一个小技术点：定时任务调度。定时任务也是我们在项目中经常使用的技术之一，接下来的 3 章中，我们会分别讲解单体应用的定时任务方案 Quartz ，和分布式应用中可以引入的分布式定时任务方案 XXL-job 。

本章我们先来玩一下适合单体应用使用的定时任务方案 Quartz ，其实这个工具在很早之前就有了，而且使用的场景还挺频繁的，本章我们主要讲解它的几种使用方式，以及与 SpringBoot 整合的时候底层自动装配都做了什么。

## 1. Quartz的使用

Quartz 是 OpenSymphony 这个开源组织下的定时调度框架，它可以整合到几乎任何 Java 项目中执行定时调度任务，包括它还支持一些企业级特性，包括 JTA 事务、集群的支持。

<!--more-->

本来我们可以依靠 jdk 中原生的 Timer 来实现定时任务，为什么还要用 Quartz 呢？那是因为这个家伙支持 cron 表达式，这个表达式实在是太好用了！下面我们先回顾 cron 表达式都怎么用。

### 1.1 Cron表达式

Cron 表达式通常以 6 个或 7 个域组成，每两个域之间用空格隔开，每个域表达的含义不同，具体如下：

【seconds minutes hours day-of-month month day-of-week year】

其中 year 可有可无。

至于每个域中都能写什么呢？如果按照 Quartz 官方文档中写的详细写法，估计小伙伴们都要看吐了，倒不如小册来通过一些实际例子来演示。

- `0 0 10,14,16 * * ?` ：每天上午 10 点，下午 2 点、4 点
- `0 0/30 9-17 * * ?` ：上午 9 点到下午 5 点内每半小时
- `0 0 12 ? * WED` ：表示每个星期三中午 12 点
- `0 0 12 ? * WED` ：表示每个星期三中午 12 点
- `0 0 12 * * ?` ：每天中午 12 点触发
- `0 15 10 ? * *` ：每天上午 10:15 触发
- `0 15 10 * * ? *` ：每天上午 10:15 触发
- `0 15 10 * * ? 2022` ：2022 年的每天上午 10:15 触发
- `0 * 14 * * ?` ：在每天下午 2 点到下午 2:59 期间的每 1 分钟触发
- `0 0/5 14 * * ?` ：在每天下午 2 点到下午 2:55 期间的每 5 分钟触发
- `0 0/5 14,18 * * ?` ：在每天下午 2 点到 2:55 期间和下午 6 点到 6:55 期间的每 5 分钟触发
- `0 0-5 14 * * ?` ：在每天下午 2 点到下午 2:05 期间的每 1 分钟触发
- `0 10,44 14 ? 3 WED` ：每年三月的星期三的下午 2:10 和 2:44 触发
- `0 15 10 ? * MON-FRI` ：周一至周五的上午 10:15 触发
- `0 15 10 15 * ?` ：每月 15 日上午 10:15 触发
- `0 15 10 L * ?` ：每月最后一日的上午 10:15 触发
- `0 15 10 ? * 6L` ：每月的最后一个星期五上午 10:15 触发
- `0 15 10 ? * 6L 2022-2023` ：2022 年至 2023 年的每月的最后一个星期五上午 10:15 触发
- `0 15 10 ? * 6#3` ：每月的第三个星期五上午 10:15 触发

上面的例子已经把 Cron 表达式的编写方式都差不多列举了，至于每个是什么含义，对比一下后面的解释，想必即便是不会写的小伙伴，照着葫芦画瓢也应该能写了。

### 1.2 SpringBoot整合Quartz

单独使用 Quartz 配合 Cron 表达式的话，本身有些费劲，编写的代码也相对费劲。SpringBoot 整合 Quartz 后，那个编码难度可以说是急剧降低。下面我们赶紧来试验一下。

##### 1.2.1 创建工程导入依赖

我们先来创建工程，依然是 Maven 工程，工程名为 `spring-boot-integration-09-quartz` ，SpringBoot 整合 Quartz 的场景启动器依赖就叫 `spring-boot-starter-quartz` ，我们把它导入即可（当然也要导入 WebMvc 的依赖）。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
</dependencies>
```

##### 1.2.2 编写代码测试效果

SpringBoot 的工程中使用 Quartz 定时任务特别简单，只需要在需要定时调度的方法上标注一个 `@Scheduled` 注解，并指定 cron 表达式即可，下面我们就来简单编写一个 `ScheduleService` ，并编写 `test` 方法打印一个简单的字符串，在方法上标注 `@Scheduled` 注解，并且指定这个方法每 10 秒触发一次。

```java
@Service
@Slf4j
public class ScheduleService {

    @Scheduled(cron = "0/10 * * * * *")
    public void test() {
        log.info("ScheduleService test invoke ......");
    }
}

```

当然，为了能直观地反映出方法被执行的时机，我们使用 Logger 日志的方式打印，这样在控制台上就能清楚地看到具体的时间了。

要想让 `@Scheduled` 注解生效，我们还需要在 SpringBoot 的主启动类（或任意一个配置类）上标注 `@EnableScheduling` 注解，代表我们要启用基于注解的定时任务调度。

```java
@SpringBootApplication
@EnableScheduling
public class QuartzApp {

    public static void main(String[] args) {
        SpringApplication.run(QuartzApp.class);
    }
}

```

如此编写完毕后，我们就可以启动工程了，稍等片刻后，控制台就会打印 Logger 日志，而且看打印的时间也知道，它触发的时机的确是每 5 秒执行一次。

```ini
2023-03-16 20:49:20.004  INFO 95968 --- [   scheduling-1] o.clxmm.quartz.service.ScheduleService   : ScheduleService test invoke ......
2023-03-16 20:49:30.004  INFO 95968 --- [   scheduling-1] o.clxmm.quartz.service.ScheduleService   : ScheduleService test invoke ......
2023-03-16 20:49:40.004  INFO 95968 --- [   scheduling-1] o.clxmm.quartz.service.ScheduleService   : ScheduleService test invoke ......
2023-03-16 20:49:50.004  INFO 95968 --- [   scheduling-1] o.clxmm.quartz.service.ScheduleService   : ScheduleService test invoke ......

```

这样我们就实现了最基本的定时任务调度。

## 2. 动态定时任务管理

如果只是像上面那样使用 Quartz ，那小册这一章也太水了吧（阿熊震怒），我们还是得讲点实用的。各位思考一下，像上面那样的使用方式，有一个什么问题？

哪些方法被调度，是不是硬编码在项目中了！

那如果我们需要动态的增减定时任务，那应该怎么办呢？Quartz 支持我们这么干吗？哎，还真能！其实我们如果稍微了解一下 Quartz 的模型，自然就知道如何处理了。

### 2.1 Quartz的内部结构

Quartz 的内部是由 3 个核心 API 构成的，它们分别是：

- `Job` ：任务模型，一个 `Job` 对象的内部封装了一个真正执行的业务逻辑；
- `Trigger` ：任务触发器，它可以触发具体的任务；
- `Scheduler` ：任务调度器，这个 API 将 `Job` 和 `Trigger` 建立起联系，并以此完成实际的任务调度。

看上去这三个 API 我们在使用时都没有涉及到，那是因为 SpringFramework 的 schedule 模块帮我们把 Quartz 又封装了一层，使得我们在使用定时任务时，只需要编写方法 + 标注注解即可，内部驱使做的事情其实就是这三个 API 的实现类做的事情。

既然我们要制作动态的定时任务，那我们就得实际操作这三个 API 了，下面我们先来演示最简单的动态定时任务。

### 2.2 基于内存的动态定时任务

##### 2.2.1 自定义定时任务

如果我们需要自定义定时任务，首先需要先自己编写对应的定时任务实现类，也就是 `Job` 接口的实现类。

```java
@Slf4j
public class SimpleJob implements Job {
    @Override
    public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        log.info("简单任务执行");
    }
}
```

此处我们为了演示方便 + 快速实现，所以我们在这里面直接用 `Logger` 打印了一个日志。

如果我们需要制作多个定时任务，那对应的就应该写多个 `Job` 接口的实现类。

##### 2.2.2 手动添加定时任务

有了定时任务后，我们就可以操作 API 来告诉 Quartz ，我们需要动态注册一个定时任务。

既然是 Web 项目，那我们可以直接编写一个 Controller ，并声明一个方法：

```java
@RestController
public class DynamicScheduleController {
    
    @GetMapping("/addSchedule")
    public String addSchedule() throws SchedulerException {
        // ????
        return "success";
    }
}
```

方法里面写什么呢？这个写法其实是固定的，包含这么 3 步：

1）创建 `JobDetail` ，2）创建 `Trigger` ；3）使用 `Scheduler` 将 `JobDetail` 和 `Trigger` 结合起来完成调度。

其实这样看来，上面我们说的 Quartz 内部核心的 3 个 API ，在这里面刚好全部都用得上。那下面我们就来简单写一个示例代码：

```java
@Autowired
private Scheduler scheduler;

@GetMapping("/addSchedule")
public String addSchedule() throws SchedulerException {
  int random = ThreadLocalRandom.current().nextInt(1000);
  // 1. 创建JobDetail
  JobDetail jobDetail = JobBuilder.newJob(SimpleJob.class)
    .withIdentity("test-schedule" + random, "test-group").build();
  // 2. 创建Trigger，并指定每3秒执行一次
  CronScheduleBuilder cron = CronScheduleBuilder.cronSchedule("0/3 * * * * ?");
  Trigger trigger = TriggerBuilder.newTrigger().withIdentity("test-trigger" + random, "test-trigger-group")
    .withSchedule(cron).build();
  // 3. 调度任务
  scheduler.scheduleJob(jobDetail, trigger);
  return "success";
}
```

注意上面代码的几个步骤，走下来刚好跟上面我们描述的一致。有两个细节需要我们注意：1）第一步创建 `JobDetail` 的时候要指定定时任务实现类的类型，所以各位可以发现，如果我们不做第一步的话，这里就卡住了；2）第二步创建 `Trigger` 的时候需要传入一个 `Schedule` 对象（注意最后没有 r ），这个对象可以是基于 Cron 表达式的 `CronScheduleBuilder` ，还可以是原生的 `SimpleScheduleBuilder` （不过我们不用罢了），另外使用 Cron 表达式构造 `CronScheduleBuilder` 的时候，它的语法要比使用 SpringFramework 时稍严格一些，所以在上面的 Cron 表达式中，我把最后一个符号由 * 换成了 ? 。

##### 2.2.3 测试效果

下面我们来试验一下效果如何，启动工程，在浏览器上发送 [http://localhost:8080/addSchedule](127.0.0.1:8080/addSchedule) 触发定时任务创建，并观察控制台的日志输出，可以发现这个新的动态任务创建完成后，的确是每 3 秒执行一次，说明效果已经正确生效了。

```ini
023-03-16 20:58:12.002  INFO 96094 --- [eduler_Worker-6] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:15.004  INFO 96094 --- [eduler_Worker-7] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:18.002  INFO 96094 --- [eduler_Worker-8] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:21.003  INFO 96094 --- [eduler_Worker-9] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:24.005  INFO 96094 --- [duler_Worker-10] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:27.004  INFO 96094 --- [eduler_Worker-1] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
2023-03-16 20:58:30.004  INFO 96094 --- [eduler_Worker-2] org.clxmm.quartz.job.SimpleJob           : 简单任务执行
```

### 2.3 基于数据库的动态定时任务

基于内存的动态定时任务还有一个比较恼火的问题：每次应用重启后，之前添加的那些定时任务就全丢了！那怎么样能保证定时任务重启后还能正常执行呢？

一个最容易想到的办法：将定时任务的信息保存到外部，每次应用启动的时候读取这个外部存储介质，并动态添加定时任务就可以了。而常用的介质当然是数据库咯，所以下面我们就来试一下把定时任务的定义信息保存到数据库中。

##### 2.3.1 改变配置

SpringBoot 在整合 Quartz 的时候特意帮我们考虑到了与 jdbc 整合的场景，它给我们提供了两个非常有用的配置，我们需要配置一下：

```properties
# 设置将定时任务的信息保存到数据库
spring.quartz.job-store-type=jdbc
# 每次应用启动的时候都初始化数据库表结构
spring.quartz.jdbc.initialize-schema=always
```

两个配置项都还算好理解吧，但是小心有坑：如果 `spring.quartz.jdbc.initialize-schema` 设置为 always 的话有个问题：每次重启应用的时候，跟 Quartz 相关的表会被删除重建！所以为了避免表被重复创建，我们可以提前创建，也可以先用 always 模式执行一次后，再调为 never （不初始化）即可。

当然，整合 jdbc 时，各位不要忘记导入 MySQL 的驱动，以及 jdbc 的场景启动器，还有配置数据源。这些简单操作各位自己完成即可，小册就不再啰里啰嗦的贴代码了。

##### 2.3.2 测试效果

准备工作都完成后，我们就可以直接启动工程观察效果了。刚启动的时候只会触发第 1 节中我们使用注解的方式编写的定时任务，而当我们在浏览器发送 `/addSchedule` 请求后，来到数据库中可以发现，有几张表中多了一行数据，我们点开 `qrtz_job_details` 表，可以看到我们编写的那个定时任务的定义信息已经被保存到数据库了。

![](./img/202303/33croe.png)

如果我们把 `spring.quartz.jdbc.initialize-schema` 的值改为 `never` 之后重启工程，可以发现我们自己编写的任务照样能正常触发，这就说明 Quartz 结合数据库的动态定时任务已经成功实现了。

> 如果小伙伴有接触过原生的 Quartz 对接数据库，那的确是有点麻烦的，可见 SpringBoot 帮我们做的事情之多。

##### 2.3.3 暂停与恢复

定时任务创建出来了，如果我们需要临时暂停定时任务的执行，亦或者恢复暂停的任务，同样可以操作 `Scheduler` 的 API 来实现，对应的操作代码如下所示。

```java
    @GetMapping("/pauseSchedule")
    public String pauseSchedule(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
        // 获取定时任务
        JobDetail jobDetail = scheduler.getJobDetail(jobKey);
        if (jobDetail == null) {
            return "error";
        }
        scheduler.pauseJob(jobKey);
        return "success";
    }
    
    @GetMapping("/remuseSchedule")
    public String remuseSchedule(String jobName, String jobGroup) throws SchedulerException {
        JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
        // 获取定时任务
        JobDetail jobDetail = scheduler.getJobDetail(jobKey);
        if (jobDetail == null) {
            return "error";
        }
        scheduler.pauseJob(jobKey);
        return "success";
    }
```

##### 2.3.4 移除定时任务

如果有些定时任务我们不需要了，可以操作 Scheduler 的 API 来移除。与前面 3 个操作不同，移除定时任务需要先将定时任务的触发器停止，之后移除掉这个触发器，最后删除定时任务，一共要操作 3 个不同的方法。

```java
    @GetMapping("/removeSchedule")
    public String removeSchedule(String jobName, String jobGroup, String triggerName, String triggerGroup) throws SchedulerException {
        TriggerKey triggerKey = TriggerKey.triggerKey(triggerName, triggerGroup);
        JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
        Trigger trigger = scheduler.getTrigger(triggerKey);
        if (trigger == null) {
            return "error";
        }
        // 停止触发器
        scheduler.pauseTrigger(triggerKey);
        // 移除触发器
        scheduler.unscheduleJob(triggerKey);
        // 删除任务
        scheduler.deleteJob(jobKey);
        return "success";
    }
```

## 3. 自动装配

----

本章的最后，我们还是按照老套路，从底层扒一扒 SpringBoot 整合 Quartz 的源码实现。但是与之前的场景不同，我们在整合 Quartz 时，如果只是整合上，不标注 `@EnableScheduling` 注解，定时任务是没有办法触发的，所以我们首先要看的是 `@EnableScheduling` 这个注解。

### 3.1 @EnableScheduling

```java
@Import(SchedulingConfiguration.class)
public @interface EnableScheduling {

}
```

不出意外，`@EnableXXX` 系注解都是使用 `@Import` 注解来导入注解配置类，这里它导入的配置类是 `SchedulingConfiguration` ，我们直接进入其中即可。

```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {

    @Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```

合着配置类中就注册了一个 `ScheduledAnnotationBeanPostProcessor` ，从类名上就能看得出来，它就是用来检测类中是否有包含 `@Scheduled` 注解标注的方法，并封装为定时任务的。它的处理逻辑我们也可以进去看一看。

### 3.2 ScheduledAnnotationBeanPostProcessor

`ScheduledAnnotationBeanPostProcessor` 本身是一个 `BeanPostProcessor` ，既然它具备收集 Bean 所属类的注解信息，那 `postProcessAfterInitialization` 方法必定是我们研究的重点。除了这个方法之外，它还实现了 `ApplicationListener` 接口，监听 `ContextRefreshedEvent` 事件，所以还有一个要关注的方法是 `onApplicationEvent` 。我们分开来看。

#### 3.2.1 postProcessAfterInitialization

```java
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (bean instanceof AopInfrastructureBean || bean instanceof TaskScheduler ||
				bean instanceof ScheduledExecutorService) {
			// Ignore AOP infrastructure such as scoped proxies.
			return bean;
		}

		Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
		if (!this.nonAnnotatedClasses.contains(targetClass) &&
				AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {
			Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
					(MethodIntrospector.MetadataLookup<Set<Scheduled>>) method -> {
						Set<Scheduled> scheduledAnnotations = AnnotatedElementUtils.getMergedRepeatableAnnotations(
								method, Scheduled.class, Schedules.class);
						return (!scheduledAnnotations.isEmpty() ? scheduledAnnotations : null);
					});
			if (annotatedMethods.isEmpty()) {
				this.nonAnnotatedClasses.add(targetClass);
				if (logger.isTraceEnabled()) {
					logger.trace("No @Scheduled annotations found on bean class: " + targetClass);
				}
			}
			else {
				// Non-empty set of methods
				annotatedMethods.forEach((method, scheduledAnnotations) ->
						scheduledAnnotations.forEach(scheduled -> processScheduled(scheduled, method, bean)));
				if (logger.isTraceEnabled()) {
					logger.trace(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
							"': " + annotatedMethods);
				}
			}
		}
		return bean;
	}
```

虽然整段源码读起来有点乱，但是从中间的 if 结构判断中就能看出，它检查 Bean 所属的 Class 中是否包含 `@Scheduled` 和 `@Schedules` 注解，并当存在时将这些方法都收集起来，执行中间偏下的 `processScheduled` 方法，这个方法执行完毕后，底层就会封装一个 `CronTask` 对象，留着在下面的 `onApplicationEvent` 方法中使用。

#### 3.2.2 onApplicationEvent

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (event.getApplicationContext() == this.applicationContext) {
        finishRegistration();
    }
}
```

下面我们就来到 `onApplicationEvent` 方法，当 IOC 容器刷新完毕后，它会执行下面的 `finishRegistration` 方法，而这个方法又会向下调用 `ScheduledTaskRegistrar` 的 `afterPropertiesSet` 方法，触发其初始化逻辑。

```java
private void finishRegistration() {
    // 好长的一段源码 ......
    
    this.registrar.afterPropertiesSet();
}
```

继续向下来到 `ScheduledTaskRegistrar` 中，这里面就有启用 Cron 表达式的配置了（源码的下半部分）。不过到此为止我们一直没有找到与 Quartz 相关的东西呢！这是为什么呢？

```java
public void afterPropertiesSet() {
    scheduleTasks();
}


protected void scheduleTasks() {
    if (this.taskScheduler == null) {
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    if (this.triggerTasks != null) {
        for (TriggerTask task : this.triggerTasks) {
            addScheduledTask(scheduleTriggerTask(task));
        }
    }
    if (this.cronTasks != null) {
        for (CronTask task : this.cronTasks) {
            addScheduledTask(scheduleCronTask(task));
        }
    }
    // ......
}
```

#### 3.2.3 没有quartz的原因

其实啊，当我们使用 `@EnableScheduing` 注解启用定时调度时，底层其实用的是 SpringFramework 原生封装的一套定时调度机制，压根就跟 Quartz 没有任何关系，即使是原生的 SpringFramework 应用，也是完全可以使用的。

那为什么 SpringBoot 在整合定时调度时，究竟都做了哪些跟 Quartz 相关的配置呢？

### 3.3 QuartzAutoConfiguration

别忘了，SpringBoot 之所以能直接支持内存的定时任务和数据库的定时任务，就是底层自动装配的功劳，而这个自动装配就是与 Quartz 相关的。下面我们也来看一看 `QuartzAutoConfiguration` 都做了什么。

#### 3.3.1 条件装配与暗示

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Scheduler.class, SchedulerFactoryBean.class, PlatformTransactionManager.class })
@EnableConfigurationProperties(QuartzProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class,
		LiquibaseAutoConfiguration.class, FlywayAutoConfiguration.class })
public class QuartzAutoConfiguration
```

从类头上标注的条件装配注解上就能看出，这个类想生效，就必须要在项目的 classpath 下存在 Quartz 的 `Scheduler` 、SpringFramework 与 Quartz 整合的工厂 Bean `SchedulerFactoryBean` ，以及与 jdbc 相关的事务管理器 `PlatformTransactionManager` 。这也就暗示我们：只有当项目中有与数据库的交互时，才有可能用到 Quartz ，否则只用 SpringFramework 自己封装的那一套就足够用了。

那当这个类激活的时候，这个自动装配都注册了哪些 bean 对象呢？我们继续往下看。

#### 3.3.2 SchedulerFactoryBean

```java
@Bean
@ConditionalOnMissingBean
public SchedulerFactoryBean quartzScheduler(QuartzProperties properties,
        ObjectProvider<SchedulerFactoryBeanCustomizer> customizers, ObjectProvider<JobDetail> jobDetails,
        Map<String, Calendar> calendars, ObjectProvider<Trigger> triggers, ApplicationContext applicationContext) {
    SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
    SpringBeanJobFactory jobFactory = new SpringBeanJobFactory();
    jobFactory.setApplicationContext(applicationContext);
    schedulerFactoryBean.setJobFactory(jobFactory);
    // 一堆setter ......
    schedulerFactoryBean.setJobDetails(jobDetails.orderedStream().toArray(JobDetail[]::new));
    schedulerFactoryBean.setCalendars(calendars);
    schedulerFactoryBean.setTriggers(triggers.orderedStream().toArray(Trigger[]::new));
    // 注意此处有定制器的回调
    customizers.orderedStream().forEach((customizer) -> customizer.customize(schedulerFactoryBean));
    return schedulerFactoryBean;
}
```

从类名上可以很明显地看出，`SchedulerFactoryBean` 的作用就是生产 Quartz 中的那个 `Scheduler` 对象的。如果我们使用编码的形式，在项目中静态注册了一些 `JobDetail` 和 `Trigger` 类型的 Bean ，那这里会收集起来，放入 `SchedulerFactoryBean` 中，以备最终创建 `Scheduler` 时使用。

等下，好像还没看到跟数据库相关的配置吧，这也只是把 Quartz 中核心的那个 `Scheduler` 给创建出来了而已，跟数据库相关的东西在哪里呢？数据库中的那些定时任务信息又是如何被注册到 `Scheduler` 中呢？我们继续往下看。

在这个 `QuartzAutoConfiguration` 的下面还有一个内部类 `JdbcStoreTypeConfiguration` ，这里就是注册与数据库相关的组件了。

#### 3.3.3 SchedulerFactoryBeanCustomizer

```java
@Bean
@Order(0)
public SchedulerFactoryBeanCustomizer dataSourceCustomizer(QuartzProperties properties, DataSource dataSource,
        @QuartzDataSource ObjectProvider<DataSource> quartzDataSource,
        ObjectProvider<PlatformTransactionManager> transactionManager,
        @QuartzTransactionManager ObjectProvider<PlatformTransactionManager> quartzTransactionManager) {
    return (schedulerFactoryBean) -> {
        DataSource dataSourceToUse = getDataSource(dataSource, quartzDataSource);
        schedulerFactoryBean.setDataSource(dataSourceToUse);
        PlatformTransactionManager txManager = getTransactionManager(transactionManager,
                quartzTransactionManager);
        if (txManager != null) {
            schedulerFactoryBean.setTransactionManager(txManager);
        }
    };
}
```

注意观察，这是一个 `SchedulerFactoryBeanCustomizer` ，刚好跟上面我们看到的源码中，我在倒数第二行那里标注的注释相呼应了。`SchedulerFactoryBean` 在创建 `Scheduler` 之前会回调这些定制器，而刚好这注册的就是一个相应的定制器，它的作用就是拿到 `PlatformTransactionManager` 和 `DataSource` ，并与 `SchedulerFactoryBean` 完成关联。

简单的说，经过这个定制器的定制后，`Scheduler` 的内部就组合了我们的 `DataSource` ，这样后面就具备与数据库交互的能力了。

#### 3.3.4 QuartzDataSourceInitializer

```java
@Bean
@ConditionalOnMissingBean
public QuartzDataSourceInitializer quartzDataSourceInitializer(DataSource dataSource,
        @QuartzDataSource ObjectProvider<DataSource> quartzDataSource, ResourceLoader resourceLoader,
        QuartzProperties properties) {
    DataSource dataSourceToUse = getDataSource(dataSource, quartzDataSource);
    return new QuartzDataSourceInitializer(dataSourceToUse, resourceLoader, properties);
}
```

最后一个注册的组件，从类名上也能看得出来，它是一个加载器，得益于上面定制器的处理，这个加载器就可以拿 `DataSource` 将 Quartz 需要的那些数据库表创建出来，而创建的方案无疑是靠 SQL 脚本来执行。

怎么读取和加载 SQL 脚本呢？我们需要来到 `QuartzDataSourceInitializer` 的父类 `AbstractDataSourceInitializer` 中，这个家伙实现了 `InitializingBean` 接口，对应的 `afterPropertiesSet` 方法中调用了下面的 `initialize` 方法，虽然这个方法的源码看上去有些不是很好懂，但关键的部分还是很突出的：`populator.addScript(this.resourceLoader.getResource(schemaLocation));` 这句话可以从 classpath 中把对应的 SQL 脚本取出，并在最后使用 `DatabasePopulatorUtils` 执行这些脚本，以此就可以完成数据库表的初始化。

```java
public void afterPropertiesSet() {
    initialize();
}

protected void initialize() {
    if (!isEnabled()) {
        return;
    }
    ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
    String schemaLocation = getSchemaLocation();
    if (schemaLocation.contains(PLATFORM_PLACEHOLDER)) {
        String platform = getDatabaseName();
        schemaLocation = schemaLocation.replace(PLATFORM_PLACEHOLDER, platform);
    }
    populator.addScript(this.resourceLoader.getResource(schemaLocation));
    populator.setContinueOnError(true);
    customize(populator);
    DatabasePopulatorUtils.execute(populator, this.dataSource);
}
```

到这里，与 Quartz 相关的组件就全部列举完毕了，可以发现 SpringBoot 秉持的原则是：能不用 Quartz 就不用（自己封装的够用），只有我们真的需要，它才会动用 Quartz 的东西，这个里面小伙伴们也要理解了。

【单机的 Quartz 自然够用，但是分布式 / 微服务的开发场景中这玩意还够用吗？有没有合适的分布式场景中使用的集中式统一的任务调度中心呢？当然有，下一章我们就来介绍和使用 xxl-job 】

