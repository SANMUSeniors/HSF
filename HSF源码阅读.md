# HSF源码阅读
## HSF各组成之间的关系
![](https://images2018.cnblogs.com/blog/633531/201805/633531-20180511234139265-75892407.png)

### 1 服务提供者注册与发布
```
<bean id="hsfTestService"
        class="com.test.service.impl.HsfTestServiceImpl" />
    <bean class="com.taobao.hsf.app.spring.util.HSFSpringProviderBean"
        init-method="init">
        <property name="serviceName" value="hsfTestService" />
        <property name="target" ref="hsfTestService" />
        <property name="serviceInterface">
            <value>com.test.service.HsfTestService
            </value>
        </property>
        <property name="serviceVersion">
            <value>${hsf.common.provider.version}</value>
        </property>
    </bean>
````

首先服务发布初始化bean，HSFSpringProviderBean实现了Spring的3个接口，将HSF的publish和Spring容器的生命周期绑定在一起。

1）InitializingBean，实现afterPropertiesSet接口，在init方法之前调用，执行服务发布的初始化信息
```
@Override
public void afterPropertiesSet() throws Exception {
    init();
}　
```
2）ApplicationContextAware，在该方法会在Spring容器加载Bean之前执行，这里面最关键的就是设定了isInSpringContainer=true。它对后面的初始化有什么用呢？一般我们在配置HSFSpringProviderBean都会指定它的init-method，也就是这个HSFSpringProviderBean加载完成后执行的一个初始化方法，这个初始化方法中就是判断isInSpringContainer的值，如果为true,则不会在这里执行publish操作

3）ApplicationListener，这个方法会在所有的Bean初始化完成以后被Spring回调，这就保证了当所有的Bean初始化完成（包括各种设值注入和init方法执行）后，判断是事件ContextRefreshedEvent来执行publish方法，Spring销毁时，判断ContextClosedEvent事件，执行服务的关闭

首先看，init方法
```
public void init() throws Exception {
        // 避免被初始化多次
        if (!providerBean.getInited().compareAndSet(false, true)) {
            return;
        }
        LoggerInit.initHSFLog();

        SpasInit.initSpas();
        providerBean.checkConfig();
        publishIfNotInSpringContainer();
    }

    private void publishIfNotInSpringContainer() {
        if (!isInSpringContainer) {
            LOGGER.warn("[SpringProviderBean]不是在Spring容器中创建, 不推荐使用");
            providerBean.publish();
        }
```
从代码中很明显的看到服务发布providerBean.publish()，先来看大致类图，类图中有些不是很关键的先省略了：
![](https://images2018.cnblogs.com/blog/633531/201807/633531-20180726155321143-1933793797.png)

整个服务发布的流程，归纳如下：

* 服务初始化，首先需要有一个提供服务的service实现类和接口；

* 初始化HSFSpringProviderBean，从配置文件获取服务名称、接口、实现类、版本等等；

* providerBean是HSFApiProviderBean在HSFSpringProviderBean中的变量，HSFSpringProviderBean会将从配置文件获取的服务名称、接口、实现类、版本等等赋值给providerBean；

* providerBean中有个服务实体类ServiceMetadata，providerBean会将服务发布的所有信息放在这里，如接口、实现类、版本等等，在整个发布过程中，ServiceMetadata是所有对象之间的传输对象；

* 这里先来解释一下为什么有HSFSpringProviderBean和HSFApiProviderBean，其实两个可以合并成一个，但是为什么要分开呢？我的理解是对于不同环境的不同实现，比如现在用的是spring环境，那就需要有个spring适配类HSFSpringProviderBean来获取配置信息，假如是其他环境那么就会有另一个适配类，最终把信息统一转成给HSFApiProviderBean，HSFApiProviderBean是来具体操作实现；
* 当执行providerBean.publish()时，会调用ProcessService的publish方法，具体实现类是ProcessComponent；
* 发布的具体流程就是ProcessComponent里：
    + 第一步，调用rpcProtocolService来注册发布RPC服务，首先启动一个hsf server服务，其实就是一个netty的服务端，通过与配置中心连接，发布自己的服务。处理RPC请求就是建立一个netty链接，解析请求参数返回处理结果。然后在server本地发布一个线程池，每一个服务都会申请一个线程池，当请求过来时从线程池获取executor进行执行并返回，最后将provider封装为ProviderServiceModel，注册到HSF Server上
    + 第二步，检查单元化发布，就unitService在发布前检查是中心发布还是单元发布，对ServiceMetadata设置不同的发布路由；
    + 第三步，通过metadataService将ServiceMetadata发布到ConfigServer上，供服务的消费方能从configserver获取到服务提供方的信息
    + 第四步，通过metadataInfoStoreService将ServiceMetadata保存到redis供服务治理或者其他用途。

### 2 服务消费者的订阅和被推送
```
<bean id="hsfTestService" class="com.taobao.hsf.app.spring.util.HSFSpringConsumerBean" init-method="init">
    <property name="interfaceName" value="com.test.service.hsfTestService"/>
    <property name="version" value="1.0.0.daily"/>
</bean>
````

![](https://images2018.cnblogs.com/blog/633531/201807/633531-20180726173258156-414821133.png)

服务的订阅流程：

* 服务初始化，首先需要引入服务接口相关的pom，然后写配置文件；
* 将需要被调用的服务注册成spring bean，即上面配置文件中的内容。
    + 这里用到了动态代理，通过类图我们可以看到HSFSpringConsumerBean实现了FactoryBean；
    + FactoryBean：是一个Java Bean，但是它是一个能生产对象的工厂Bean，通过getObject方法返回具体的bean，在spring bean实例化bean的过程中会去判断是不是FactoryBean，如果不是就返回bean，否则返回FactoryBean生产的bean，具体同学们可以去看AbstractBeanFactory的doGetBean方法，里面会调用getObjectForBeanInstance方法，这个方法里有具体实现；
    + HSFSpringConsumerBean实现了FactoryBean，那么getObject方法具体返回了什么呢？怎么返回的呢？
```
@Override
public Object getObject() throws Exception {
    return consumerBean.getObject();
}
```

从代码看得出是调用了consumerBean（HSFApiConsumerBean）的getObject方法返回的，那么我们再来看getObject方法

```
public Object getObject() throws Exception {
    return metadata.getTarget();
}
```
这个方法返回的是metadata（ServiceMetadata）的target，那么target是怎么获取的呢？下面重点说明；
* HSFSpringConsumerBean的init方法调用了consumerBean（HSFApiConsumerBean）的init方法，我们来看consumerBean里init方法的某一段代码

```
ProcessService processService = HSFServiceContainer.getInstance(ProcessService.class);
try {
    metadata.setTarget(processService.consume(metadata));
    LOGGER.warn("成功生成对接口为[" + metadata.getInterfaceName() + "]版本为[" + metadata.getVersion() + "]的HSF服务调用的代理！");
} catch (Exception e) {
    LOGGER.error("", "生成对接口为[" + metadata.getInterfaceName() + "]版本为[" + metadata.getVersion()
            + "]的HSF服务调用的代理失败", e);
    // since 2007，一旦初始化异常就抛出
    throw e;
}
int waitTime = metadata.getMaxWaitTimeForCsAddress();
if (waitTime > 0) {
    try {
        metadata.getCsAddressCountDownLatch().await(waitTime, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        // ignore
    }
}
```
* 这一段代码包含了动态代理对象的具体生成和服务订阅以及服务信息接收
* 先说了一下代码逻辑，服务的订阅和服务信息的接收（被推送）在processService中执行，动态代理对象在processService中生成，下面的wait是用来等目标服务信息的推送（当收到订阅的目标具体服务实现，接下来的调用过程才能走通）

    + 首先去缓存中找是否之前target有生成，有就返回；
    + 初始化寻址服务，调用hookService.preConsume；
    + 生成对目标服务的寻址对象后，然后通过java Proxy生成代理对象；
    + 完成HSF消费端的本地初始化后，然后从HSF治理中心订阅服务信息（返回的可调用地址）；
        + 订阅路由规则和机房规则；
        + 订阅服务，首先根据订阅信息生成一个订阅登记表；其次，向订阅中心订阅服务；最后，注册订阅数据监听器，并立即执行地址刷新任务；
        + 订阅notify，支持消费端回调；
    + 保存客户端metadata到redis，返回target。

### 3 服务消费者发起调用

服务信息已经注册发布，客户端也获取到了服务的调用地址，接下去就是调用就行，调用就是真正的rpc请求了，hsf的rpc是通过netty实现的。

![](https://images2018.cnblogs.com/blog/633531/201807/633531-20180730121822169-2088970630.png)

当服务消费方发起调用时，对代理对象的调用，都会触发到HSFServiceProxy的invoke方法，该方法是服务调用的入口方法，invoke会调用trueInvoke方法：

* trueInvoke方法首先获取一个线程池的大小，如果没有，将同步分配一个线程，并调用rpc的invoke方法

* 在trueInvoke里调用RPCProtocolTemplateService，在这里封装HSFRequest，地址路由的获取、检查，监测信息的埋点，日志的处理等，然后执行具体的invoke0调用；
* invoke0方法调用流程
    + 组装HSFRequest对象，封装目标方法名称、方法参数、方法参数泪下等信息
    + 根据需要获取目标方法的URL，调用地址。不同情况选择不同的调用地址；
    + 如果是广播模式，那么对应每一个地址都会发送一次调用
    + 检查是否是本地调用；获取超时时间；根据调用类型获取不同的RPC调用，HSF主要包含3种RPC调用，CallbackInvokeComponent（回调的方式）、FutureInvokeComponent（future方式）、SyncInvokeComponent（同步调用）
+ 对于SyncInvokeComponent调用方式，
    + 创建一个Netty客户端链接
    + 向客户端提交一个HSFRequest请求，得到一个future对象
    + 通过future.get获取返回值，并通过客户端传入的timeout控制超时时间，get方法在超时或者获取结果之前是阻塞的，所以是同步调用方式
    + 将获取到的结果放在HSFResponse对象中，完成调用并返回。如果有超时或者异常，则返回异常；
### 4 服务提供方处理请求
provider端启动一个NettyServer用于接收请求，那么一次请求的处理就是从NettyServer收到请求开始的。在NettyServer的start代码中，注册了一个serverHandler，用来处理客户端的请求。
+服务端会启动nettyServer，具体由NettyServerHandler来处理所有rpc请求；
+ NettyServerHandler中定义了HandleRequest方法，用来处理所有请求，当一个请求到来，handleRequest完成了以下几件事，1）根据请求类型找到对应的ServerHandler，可能是HSF Dubbo等
+ 如果没有分配线程池，则由serverHandler来同步处理请求
+ 如果分配了线程池，则封装一个HandlerRunnable对象，来异步处理请求；
+ 无论同步还是异步，都是由ServerHandler的handleRequest来处理请求，这个方法将请求对象和连接封装成一个ServerOutput对象，交由RpcRequestProcessor来处理
    + 解析请求参数
    + 在本地查找对应的ProviderServiceModel
    + 根据方法名和参数类型查找ProviderServiceModel
    + 反射的方式调用该方法，得到返回值并封装在HSFReponse中
    + 发送Response对象
![](https://images2018.cnblogs.com/blog/633531/201807/633531-20180730152654494-579646748.png)

### 5 服务消费者获取结果
在调用SyncInvokeComponent的invoke方法中，使用future.get获取调用结果，getResponseObject方法负责从返回对象中解析结果


    




    
