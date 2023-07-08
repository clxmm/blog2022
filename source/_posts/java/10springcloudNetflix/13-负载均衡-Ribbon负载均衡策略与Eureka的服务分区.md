---
title: 13-负载均衡-Ribbon负载均衡策略与Eureka的服务分区.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

上一章咱了解了 Ribbon 的整体服务调用的负载均衡全流程，当时咱使用的策略是轮询。Ribbon 默认就提供了7种策略，下面就几种重要的负载均衡策略进行原理解析。

## 1. Ribbon负载均衡策略解析

<!--more-->

### 1.0 默认的7种策略

1. `RoundRobinRule` ：轮询
2. `RandomRule` ：随机
3. `AvailabilityFilteringRule` ：会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，还有并发的连接数量超过阈值的服务，然后对剩余的服务列表按照轮询策略进行访问
4. `WeightedResponseTimeRule` ：根据平均响应时间计算所有服务的权重，响应时间越快服务权重越大被选中的概率越高
   - 整体服务刚启动时如果统计信息不足，则使用 `RoundRobinRule` 策略，等统计信息足够，会切换到 `WeightedResponseTimeRule`
5. `RetryRule` ：先按照 `RoundRobinRule` 的策略获取服务，如果获取服务失败则在指定时间内会进行重试，获取可用的服务
   - 可以类比一个人的思维，如果连续失败几次之后，就不会再去失败的微服务处请求
6. `BestAvailableRule` ：会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
7. `ZoneAvoidanceRule` ：默认规则，复合判断服务所在区域（region）的性能和服务的可用性选择服务器

### 1.1 RoundRobinRule-轮询

上一章咱已经完整的看过一遍了，这里我把源码贴上，小伙伴们扫一遍加深印象即可。

```java
public Server choose(ILoadBalancer lb, Object key) {
    // null check ......
    Server server = null;
    int count = 0;
    // 最多尝试执行10次轮询（考虑到有线程并发没有获取到的可能）
    while (server == null && count++ < 10) {
        // 获取所有可访问的服务
        List<Server> reachableServers = lb.getReachableServers();
        List<Server> allServers = lb.getAllServers();
        int upCount = reachableServers.size();
        int serverCount = allServers.size();

        // 只要没有可访问的服务，直接返回null
        if ((upCount == 0) || (serverCount == 0)) {
            log.warn("No up servers available from load balancer: " + lb);
            return null;
        }

        // 【核心】使用incrementAndGetModulo方法决定出轮询的下标位置并取出
        int nextServerIndex = incrementAndGetModulo(serverCount);
        server = allServers.get(nextServerIndex);

        // 如果没有取出，重试
        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        // 恰巧取出的服务又不可用，重试
        if (server.isAlive() && (server.isReadyToServe())) {
            return (server);
        }

        // Next.
        server = null;
    }

    // 如果重试次数超过10次，打印警告日志，返回null
    if (count >= 10) {
        log.warn("No available alive servers after 10 tries from load balancer: " + lb);
    }
    return server;
}
```

核心方法 `incrementAndGetModulo` ：

```java
private AtomicInteger nextServerCyclicCounter;

private int incrementAndGetModulo(int modulo) {
    for (;;) {
        // 借助AtomicInteger保证线程安全
        int current = nextServerCyclicCounter.get();
        // 取模运算实现轮询
        int next = (current + 1) % modulo;
        if (nextServerCyclicCounter.compareAndSet(current, next)) {
            return next;
        }
    }
}
```

### 1.2 RandomRule-随机

随机负载均衡策略的整体看起来很像轮询，只不过这里面的核心算法改变了而已。下面是源码的核心部分，关键部分已标注注释：

```java
public Server choose(ILoadBalancer lb, Object key) {
    // .......
    while (server == null) {
        // ......
        // 获取所有可访问的服务
        List<Server> upList = lb.getReachableServers();
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();
        if (serverCount == 0) {
            return null;
        }

        // 【核心】随机生成一个索引值
        int index = chooseRandomInt(serverCount);
        server = upList.get(index);

        // ......
    }
    return server;
}
```

核心算法 `chooseRandomInt` ：

```java
protected int chooseRandomInt(int serverCount) {
    return ThreadLocalRandom.current().nextInt(serverCount);
}
```

可见就是简单的拿 `Random` 生成随机数而已，没什么复杂的。`ThreadLocalRandom` 这个组件是 java7 出现在并发包中的随机工具类，也没什么新鲜的，了解、会用就行。

### 1.3 WeightedResponseTimeRule-权重

注意 `WeightedResponseTimeRule` 继承自 `RoundRobinRule` ，这也跟这种负载均衡的规则描述一致：没有服务被调用、总权重未初始化时会使用轮询策略。源码的核心部分：

```java
public Server choose(ILoadBalancer lb, Object key) {
    // ......
    Server server = null;
    while (server == null) {
        // get hold of the current reference in case it is changed from the other thread
        // 获取当前引用，以防其他线程更改
        List<Double> currentWeights = accumulatedWeights;
        // ......
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();
        if (serverCount == 0) {
            return null;
        }

        int serverIndex = 0;
        // last one in the list is the sum of all weights
        // 获取最后一个服务实例的权重值
        double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
        // No server has been hit yet and total weight is not initialized fallback to use round robin
        // 如果没有服务被调用，或者最后一个服务实例的权重未达到0.001时会使用轮询
        if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
            server =  super.choose(getLoadBalancer(), key);
            if (server == null) {
                return server;
            }
        } else {
            // generate a random weight between 0 (inclusive) to maxTotalWeight (exclusive)
            // 最后一个服务实例超过0.001，生成一个随机值
            double randomWeight = random.nextDouble() * maxTotalWeight;
            // pick the server index based on the randomIndex
            // 根据随机值选择对应的服务
            int n = 0;
            for (Double d : currentWeights) {
                if (d >= randomWeight) {
                    serverIndex = n;
                    break;
                } else {
                    n++;
                }
            }
            server = allList.get(serverIndex);
        }
        // ......
    }
    return server;
}
```

这个策略的算法相对复杂，核心部分的思路是下面的 else 块：每次会产生一个随机值，根据这个随机值选择对应的服务实例。当然不是简单地拿随机值去算，这里面有一套策略算法。

#### 1.3.1 权重策略算法及示例

其实在 `WeightedResponseTimeRule` 的类文档注释中有比较清楚的描述权重算法的策略：

> Rule that use the average/percentile response times to assign dynamic "weights" per Server which is then used in the "Weighted Round Robin" fashion. The basic idea for weighted round robin has been obtained from JCS The implementation for choosing the endpoint from the list of endpoints is as follows:Let's assume 4 endpoints:A(wt=10), B(wt=30), C(wt=40), D(wt=20). Using the Random API, generate a random number between 1 and10+30+40+20. Let's assume that the above list is randomized. Based on the weights, we have intervals as follows: 1-----10 (A's weight) 11----40 (A's weight + B's weight) 41----80 (A's weight + B's weight + C's weight) 81----100(A's weight + B's weight + C's weight + C's weight) Here's the psuedo code for deciding where to send the request:
>
> if (random_number between 1 & 10) {send request to A;} else if (random_number between 11 & 40) {send request to B;} else if (random_number between 41 & 80) {send request to C;} else if (random_number between 81 & 100) {send request to D;}
>
> When there is not enough statistics gathered for the servers, this rule will fall back to use RoundRobinRule.
>
> 使用 平均/百分响应时间 为每个服务器分配动态“权重”的规则，然后以“加权循环”方式使用该规则。
>
> 加权轮循的基本思想是从JCS获得的。从端点列表中选择端点的实现如下：
>
> 假设有4个端点：A（wt = 10），B（wt = 30），C（wt = 40），D（wt ＝ 20）。使用随机API，生成 1 到 (10 + 30 + 40 + 20) 之间的随机数。假设上面的列表是随机的。
>
> 根据权重计算规则，计算出的间隔如下：1 ----- 10（A的权重）11 ---- 40（A的权重 + B的权重）41 ---- 80（A的权重 + B的权重 + C的权重）81 ---- 100（A的权重 + B的权重 + C的权重 + D的权重）
>
> 这是用于决定将请求发送到哪里的伪代码：
>
> if (random_number between 1 & 10) {send request to A;} else if (random_number between 11 & 40) {send request to B;} else if (random_number between 41 & 80) {send request to C;} else if (random_number between 81 & 100) {send request to D;}
>
> 如果没有为所有的服务实例收集足够的统计信息，则此规则将退化使用 RoundRobinRule 。

文档注释中对于权重的选择描述的很清楚，但它没有描述清楚权重是如何计算出来的。这个时候咱就要往下翻，在 `WeightedResponseTimeRule` 的内部类 `ServerWeight` 中有对权重值的计算：

```java
public void maintainWeights() {
    // ......
    double totalResponseTime = 0;
    // find maximal 95% response time
    for (Server server : nlb.getAllServers()) {
        // this will automatically load the stats if not in cache
        ServerStats ss = stats.getSingleServerStat(server);
        totalResponseTime += ss.getResponseTimeAvg();
    }
    // weight for each server is (sum of responseTime of all servers - responseTime)
    // so that the longer the response time, the less the weight and the less likely to be chosen
    Double weightSoFar = 0.0;
    
    // create new list and hot swap the reference
    List<Double> finalWeights = new ArrayList<Double>();
    for (Server server : nlb.getAllServers()) {
        ServerStats ss = stats.getSingleServerStat(server);
        double weight = totalResponseTime - ss.getResponseTimeAvg();
        weightSoFar += weight;
        finalWeights.add(weightSoFar);   
    }
```

这部分的计算策略如下：先把每个服务实例的平均响应时长取出并累计，再逐个计算每个服务实例的权重。权重值为 [ 总响应时间 - 服务实例平均响应时间 ] 。

##### 1.3.1.1 权重算法示例

服务实例 A B C D 的平均响应时间为 200ms ，50ms ，20ms ，80ms ，则它们的总权重值就应该是：200+50+20+80=350，每个服务实例的权重值和对应的区间范围计算得：

|              | **A**     | B           | C           | D            |
| ------------ | --------- | ----------- | ----------- | ------------ |
| 平均响应时间 | 200ms     | 50ms        | 20ms        | 80ms         |
| 权重         | 150       | 300         |             | 330          |
| 区间范围     | [0 - 150) | [150 - 450) | [450 - 780) | [780 - 1150) |

假设一次随机到的值为 436，根据上述的区间范围，发现在 [150 - 450) 中，故选择 B 服务。

#### 1.3.2 权重的统计

咱知道这些权重的计算以及如何使用，但权重本身是怎么计算得来的呢？是谁去触发了它呢？在 `WeightedResponseTimeRule` 中还有一个内部类，它是一个定时任务用来统计、计算权重：

```java
class DynamicServerWeightTask extends TimerTask {
    public void run() {
        ServerWeight serverWeight = new ServerWeight();
        try {
            serverWeight.maintainWeights();
        } // catch ......
    }
}
```

对应的，`WeightedResponseTimeRule` 的初始化方法中有对该定时任务的调度：

```java
public static final int DEFAULT_TIMER_INTERVAL = 30 * 1000;
private int serverWeightTaskTimerInterval = DEFAULT_TIMER_INTERVAL;

void initialize(ILoadBalancer lb) {        
    if (serverWeightTimer != null) {
        serverWeightTimer.cancel();
    }
    // 定时器调度
    serverWeightTimer = new Timer("NFLoadBalancer-serverWeightTimer-" + name, true);
    serverWeightTimer.schedule(new DynamicServerWeightTask(), 0,
            serverWeightTaskTimerInterval);
    // do a initial run 初始化时先统计一次
    ServerWeight sw = new ServerWeight();
    sw.maintainWeights();

    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        public void run() {
            logger.info("Stopping NFLoadBalancer-serverWeightTimer-" + name);
            serverWeightTimer.cancel();
        }
    }));
}
```

可见，在 `WeightedResponseTimeRule` 刚被创建时，就已经初始化好这个定时任务，并且预先执行了一次权重的统计计算，之后每 30 秒统计一次。

### 1.4 ZoneAvoidanceRule-分区

关于分区，那就不得不提 Eureka 的服务分区。这部分咱单开几个整节来解释，先把服务分区搞定了再解释 `ZoneAvoidanceRule` 的分区策略。

如果小伙伴对 Eureka 服务分区已经比较熟悉，可以直接跳到第4节。

可能有些小伙伴对 “Eureka 服务分区” 这个概念还不是很清楚。下面咱先介绍服务分区的相关概念、背景和设计，再解释底层的实现。

如果小伙伴对服务分区概念已经大体了解，可以跳过第2节。

## 2. 服务分区的产生

### 2.1 服务分区的起源

如果一个大型分布式项目，它的规模足够大而且部署的区域比较多且分散，这个时候就会出现一种现象：可能同一个服务的不同实例分布在多个地区的多个网络区域中。咱又知道，不同的地区、不同的网络区域，网络质量都是不尽相同的，而相同地区、相同网络区域内的服务互相通信可靠性又比较高，这个时候就需要一种机制能**尽可能的让服务通信优先选择离自己“近”的其他服务**，进而降低网络出错率，提高整体应用的稳定性和可用性。于是，产生了服务分区。

### 2.2 服务分区的核心概念

服务分区的概念最早出现于亚马逊的 AWS ，这里面设计了两个概念：**region** 、**zone** 。这两个概念都是相对于网络环境来讲的，具体可以这样理解：

- region ：地理位置的区域，如北京市区域、河南省区域、广东省区域等
- zone ：一个地理位置中的不同网络区域，如北京一区、河南三区等

### 2.3 服务分区示例

![](./img/2023/05/cloud13-1.png)

如上图所示，图中共设计了2个 region 、4个 zone ，其中 Provider1 、Provider2 、Consumer 连接到 EurekaServer1 上，Provider3 连接到 EurekaServer2 上。如果 Consumer 要消费来自 Provider 的服务，则最好选择 Provider2 ，其次是 Provider1 ，最后才选择 Provider3 。

## 3. EurekaServer配置服务分区

为演示如何进行 EurekaServer 的服务分区，咱下面搭建一个相对简单的微服务网络。如果小伙伴已经了解如何搭建 EurekaServer 的服务分区，可以直接跳过第3章节。

搭建的示意图如下：

![](./img/2023/05/cloud13-2.png)

### 3.1 EurekaServer的搭建

EurekaServer 在配置服务分区时，需要配置当前的 region 与可使用的 zone ，以便服务注册时可以选择自己的分区。完整的两个实例配置如下：

eureka-server1 ：

```yaml
server:
  port: 9001
spring:
  application:
    name: eureka-server
eureka:
  instance:
    hostname: eureka-server-9001.com
  client:
    service-url:
      beijing1: http://eureka-server-9001.com:9001/eureka/
      beijing2: http://eureka-server-9002.com:9002/eureka/
    region: beijing
    availability-zones:
      beijing: beijing1,beijing2
```

eureka-server2 ：（注意这里配置时要把2放在前面，Eureka 会挑所有配置列表中第一个配置）

```yaml
server:
  port: 9002
spring:
  application:
    name: eureka-server
eureka:
  instance:
    hostname: eureka-server-9002.com
  client:
    service-url:
      beijing2: http://eureka-server-9002.com:9002/eureka/
      beijing1: http://eureka-server-9001.com:9001/eureka/
    region: beijing
    availability-zones:
      beijing: beijing2,beijing1
```

### 3.2 EurekaClient服务提供者的搭建

咱们使用普通的 EurekaClinet 作为服务提供者，配置方式与 EurekaServer 类似。

eureka-client1 ：

```yaml
server:
  port: 8080
spring:
  application:
    name: eureka-client
eureka:
  instance:
    instance-id: eureka-client1
    prefer-ip-address: true
    metadata-map:
      zone: beijing1
  client:
    service-url:
      beijing1: http://eureka-server-9001.com:9001/eureka/
      beijing2: http://eureka-server-9002.com:9002/eureka/
    region: beijing
    availability-zones:
      beijing: beijing1,beijing2

```

eureka-client2 ：

```yaml
server:
  port: 8081
spring:
  application:
    name: eureka-client
eureka:
  instance:
    instance-id: eureka-client2
    prefer-ip-address: true
    metadata-map:
      zone: beijing1
  client:
    service-url:
      beijing1: http://eureka-server-9001.com:9001/eureka/
      beijing2: http://eureka-server-9002.com:9002/eureka/
    region: beijing
    availability-zones:
      beijing: beijing1,beijing2

```

eureka-client3 ：

```yaml
server:
  port: 8082
spring:
  application:
    name: eureka-client
eureka:
  instance:
    instance-id: eureka-client3
    prefer-ip-address: true
    metadata-map:
      zone: beijing2
  client:
    service-url:
      beijing2: http://eureka-server-9001.com:9002/eureka/
      beijing1: http://eureka-server-9002.com:9001/eureka/
    region: beijing
    availability-zones:
      beijing: beijing2,beijing1
```

之后在每个 EurekaClient 工程中都加入一个 `ProviderController` 来暴露自己的 Eureka 实例ID：

```java
@RestController
public class ProviderController {
    @Value("${eureka.instance.instance-id}")
    private String zone;
    
    @GetMapping("/getInfo")
    public String getInfo() {
        return zone;
    }
}
```

这样服务提供者就搭建好了。

### 3.3 EurekaConsumer服务消费者的搭建

EurekaConsumer 作为服务的消费者，除了搭建基本的配置之外，还要再写一点内容来调用前面的服务提供者。

eureka-consumer ：

```yaml
server:
  port: 8090
spring:
  application:
    name: eureka-consumer
eureka:
  instance:
    instance-id: eureka-consumer
    prefer-ip-address: true
    metadata-map:
      zone: beijing2
  client:
    service-url:
      beijing1: http://eureka-server-9001.com:9001/eureka/
      beijing2: http://eureka-server-9002.com:9002/eureka/
    region: beijing
    availability-zones:
      beijing: beijing1,beijing2
```

咱也写一个 `ConsumerController` 来手动触发服务调用：

```java
@RestController
public class ConsumerController {
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/getInfo")
    public String getInfo() {
        return restTemplate.getForObject("http://eureka-client/getInfo", String.class);
    }
}
```

注意这个 `RestTemplate` 在构建时需要标注 `@LoadBalanced` 注解。

### 3.4 测试运行

将6个工程都启动起来，浏览器中输入 [http://localhost:8090/getInfo]http://localhost:8090/getInfo 后访问，发现返回的是 eureka-client3 ，一直不停的刷新访问也都是 eureka-client3 。

在IDEA中将 eureka-client3 停掉，此时 EurekaServer 收不到来自 eureka-client3 的心跳，一段时间后认定租约过期，服务实例下线。由于此时 beijing2 中已经没有其他的 eureka-client 服务，所以此时再刷新，才会发现返回 eureka-client1 和 eureka-client2 轮询出现。

这个现象想要复现可能需要一定时间（Eureka的高可用决定了要等一段时间才能与真实一致，放弃了强一致性），小伙伴们在测试时一定要有耐心等待。

## 4. 服务分区与分区负载均衡原理

咱对服务分区有了解之后大概能脑补出基本逻辑来：先在本 region 的本 zone 中寻找所有可用的服务，之后根据负载均衡算法 `ZoneAvoidanceRule` 调用即可。

### 4.1 ZoneAvoidanceRule的分区负载策略

翻开 `ZoneAvoidanceRule` 的源码，发现它并没有实现 `choose` 方法，而是它的父类 `PredicateBasedRule` 实现：

```java
public abstract AbstractServerPredicate getPredicate();

public Server choose(Object key) {
    ILoadBalancer lb = getLoadBalancer();
    // 根据事先规定的条件过滤服务实例后，在剩余的实例中轮询
    Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    if (server.isPresent()) {
        return server.get();
    } else {
        return null;
    }       
}
```

注意它的设计：`getPredicate` 为模板方法，让子类实现过滤策略。那咱又得跳回 `ZoneAvoidanceRule` 中：

```java
private CompositePredicate compositePredicate;

public AbstractServerPredicate getPredicate() {
    return compositePredicate;
} 

public ZoneAvoidanceRule() {
    super();
    ZoneAvoidancePredicate zonePredicate = new ZoneAvoidancePredicate(this);
    AvailabilityPredicate availabilityPredicate = new AvailabilityPredicate(this);
    compositePredicate = createCompositePredicate(zonePredicate, availabilityPredicate);
}
```

辗转反侧发现要获取的 `compositePredicate` 是通过 `createCompositePredicate` 方法创建而来。

```java
private CompositePredicate createCompositePredicate(ZoneAvoidancePredicate p1, 
         AvailabilityPredicate p2) {
    return CompositePredicate.withPredicates(p1, p2)
            .addFallbackPredicate(p2)
            .addFallbackPredicate(AbstractServerPredicate.alwaysTrue())
            .build();
}
```

看到这里发现不认识的东西越来越多了，咱一一来解释。

#### 4.1.1 CompositePredicate的作用

它的文档注释解释的还蛮清楚的：

> A predicate that is composed from one or more predicates in "AND" relationship. It also has the functionality of "fallback" to one of more different predicates. If the primary predicate yield too few filtered servers from the getEligibleServers(List, Object) API, it will try the fallback predicates one by one, until the number of filtered servers exceeds certain number threshold or percentage threshold.
>
> 由 “与” 关系中的一个或多个条件判断组成的复合条件。它还具有“回退”到更多不同条件之一的功能。如果主条件从 `getEligibleServers(List，Object)` API生成的筛选服务实例太少，它将尝试逐一回退已经判断过的条件，直到筛选的服务实例数量超过特定数量阈值或百分比阈值。

由此可见它的逻辑还挺复杂，它可以封装多个过滤条件，每次执行时都是先把所有预先封装好的条件都执行一遍，如果执行完毕后发现剩余的服务实例太少或者比例太少，会使用事先传入的回退过滤条件重新筛选，直到符合预先规定的最少服务实例数量、或者真的找不到返回空集合。

`CompositePredicate` 的核心方法如下：

```java
private int minimalFilteredServers = 1;
private float minimalFilteredPercentage = 0;

public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
    // 执行父类的方法，把所有的过滤条件全部执行一次
    List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
    Iterator<AbstractServerPredicate> i = fallbacks.iterator();
    // 如果过滤之后剩余的服务实例数量不到1个(即为空)，开始执行回退的过滤条件
    while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
            && i.hasNext()) {
        AbstractServerPredicate predicate = i.next();
        result = predicate.getEligibleServers(servers, loadBalancerKey);
    }
    return result;
}

```

#### 4.1.2 ZoneAvoidancePredicate的过滤规则

咱都知道，所有 `Predicate` 类型都要实现 `apply` 方法，这里面的核心过滤规则如下（已去除部分不重要的源码）：

```java
public boolean apply(@Nullable PredicateKey input) {
    if (!ENABLED.get()) {
        return true;
    }
    // zone判断和过滤
    String serverZone = input.getServer().getZone();
    // 没有标注zone的会直接返回，不会被过滤
    if (serverZone == null) {
        // there is no zone information from the server, we do not want to filter
        // out this server
        return true;
    }
    // ......
    Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, 
            triggeringLoad.get(), triggeringBlackoutPercentage.get());
    logger.debug("Available zones: {}", availableZones);
    // 预先配置中可见的zone中能找到当前服务实例，则通过过滤
    if (availableZones != null) {
        return availableZones.contains(input.getServer().getZone());
    } else {
        return false;
    }
}

```

可以总结出来 `ZoneAvoidancePredicate` 的过滤目的是：**将可用的 zone 筛出来，并过滤掉该 zone 中不可用的服务实例** 。

#### 4.1.3 AvailabilityPredicate的过滤规则

```java
public boolean apply(@Nullable PredicateKey input) {
    LoadBalancerStats stats = getLBStats();
    if (stats == null) {
        return true;
    }
    return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
}

private boolean shouldSkipServer(ServerStats stats) {
    // isCircuitBreakerTripped : 被断路器保护
    // activeConnectionsLimit : Integer.MAX_VALUE
    if ((CIRCUIT_BREAKER_FILTERING.get() && stats.isCircuitBreakerTripped()) 
            || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
        return true;
    }
    return false;
}

```

可以总结 `AvailabilityPredicate` 的过滤目的是：**将已经被断路器保护的服务、或者连接数太多的服务实例过滤掉**。

### 4.2 Debug过滤实例

将上面的服务分区实例全部启动，在 `CompositePredicate` 的 `getEligibleServers` 中打入断点。

初次Debug发现能且仅能获取一个 eureka-client 实例（注册在 beijing2 的那个）：

![](./img/2023/05/cloud13-3.png)

将 eureka-client3 停掉，一段时间后再Debug进入该位置，发现 result 为2，这时 `ZoneAvoidancePredicate` 中的 `lbStats.getAvailableZones().size() <= 1` 条件触发，直接返回，所以会出现两个来自 beijing1 的服务实例。

![](./img/2023/05/cloud13-4.png)

将 eureka-client1 与 2 全部停掉，一段时间后再Debug，发现 result 为0，因为确实已经没有任何服务实例了，此时代码会执行到 while 内部，开始倒退寻找实例，当然此时怎么找也找不到了，于是最终返回空集合，代表没有对应的服务实例了。

## 小结

1. Ribbon 默认提供7种策略，默认的策略是 `ZoneAvoidanceRule` ；
2. 轮询与随机的算法都比较简单，权重策略涉及到一个与响应时间有关的算法；
3. 服务分区中有两个核心概念：region 、zone ，服务分区能提高整体应用的可用性、降低网络延迟；
4. `ZoneAvoidanceRule` 分区策略会使用“过滤+回退”的算法筛选出符合请求的服务实例。

【到这里其实 Ribbon 的负载均衡也就解析的差不多了，下一篇咱介绍一些 Ribbon 的其他原理，这里面涉及到的内容能给小伙伴们一些启发，当然难度可能也会稍微高一些】

