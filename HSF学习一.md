# 分布式服务框架HSF学习
HSF提供的是分布式服务开发框架，taobao内部使用较多，总体来说其提供的功能及一些实现基础：
* 1.标准Service方式的RPC
    +  Service定义：基于OSGI的Service定义方式
    + TCP/IP通信：
        +  IO方式：nio,采用mina框架
        +  连接方式：长连接
        +  服务器端有限定大小的连接池
        +  WebService方式：序列化：Hessian序列化机制
* 2.软件负载体系
* 3.模块化、动态化
* 4.服务治理

## 使用：
 ```
 <dependency>  
            <groupId>com.taobao.hsf</groupId>  
            <artifactId>hsf.connector.spring</artifactId>  
            <version>xxx</version>  
 </dependency>
 ```
 而对于服务框架肯定是有服务提供者和消费者两种角色，在提供者方要做的工作包括：
 1. 将interface的代码打成Jar包，放进maven仓库中，供使用者下载使用，而具体代码实现则不需要放进jar包中，使用者只能调用，无法看见具体实现。
 2. 在对应的HSF的配置文件里，将提供的服务提供出来（基于spring的bean配置）

  ```
 <bean id="xxxServiceImpl" class="xxx.xxxServiceImpl" />  
<bean id="xxxServiceProvider"     class="com.taobao.hsf.app.spring.util.HSFSpringProviderBean" init-method="init">  
     <property name="serviceInterface">  
        <value>xxx.xxxService</value>  
     </property>  
     <property name="target">  
        <ref bean="xxxServiceImpl" />  
     </property>  
     <property name="serviceName">  
        <value>xxxService</value>  
     </property>  
     <property name="serviceVersion">  
        <value>xxx</value>  
     </property>  
     <property name="serviceGroup">  
        <value>HSF</value>  
     </property>  
</bean>  
 ```
 服务提供成功后，在HSF服务管理中心可以查看到这个HSF服务。
 而在消费者方要做的工作：
 ```
 <bean name="xxxService" class="com.taobao.hsf.app.spring.util.HSFSpringConsumerBean" init-method="init">  
     <property name="interfaceName" value="xxx.xxxService" />  
     <property name="version" value="xxx" />  
</bean>  
 ```
这样这个service就可以使用了。

HSF的缺点是其要使用指定的JBoss等容器，还需要在JBoss等容器中加入sar包扩展，对用户运行环境的侵入性大，如果你要运行在Weblogic或Websphere等其它容器上，需要自行扩展容器以兼容HSF的ClassLoader加载。 taobao有类似的其他框架Dubbo,介绍见
http://www.iteye.com/magazines/103