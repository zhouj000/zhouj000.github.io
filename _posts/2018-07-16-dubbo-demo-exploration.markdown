---
layout:     post
title:      "从dubbo-demo初探dubbo源码"
date:       2018-07-16 23:00
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - java
    - rpc
--- 

<font id="last-updated">最后更新于：2018-07-17</font>

# 准备工作

源码从[github](https://github.com/apache/incubator-dubbo)上clone

使用zookeeper作为注册中心，从[官网](https://zookeeper.apache.org/)下载

通过maven构建项目，从[官网](https://maven.apache.org/)下载

## dubbo-demo结构

+ dubbo-demo-api
	- I:DemoService
+ dubbo-demo-consumer
	- Consumer
	- dubbo-demo-consumer.xml
	- dubbo.properties
	- log4j.properties
+ dubbo-demo-provider
	- DemoServiceImpl
	- Provider
	- dubbo-demo-provider.xml
	- dubbo.properties
	- log4j.properties

# Provider

流程：registry(注册消费者)->cluster(集群处理)->dubbo.rpc(代理封装格式)->remoting(远程网络传输)  
service,config,proxy,registry,cluster,monitor,protocol,exchange,transport,serialize模块

首先修改dubbo-demo-provider.xml配置文件，将dubbo:registry设置为`zookeeper://127.0.0.1:2181`  
启动Provider的main方法。

```
public class ServiceBean<T> extends ServiceConfig<T> 
	implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {...}
```

1. ClassPathXmlApplicationContext设置xml地址。调用AbstractApplicationContext的refresh()方法。
2. finishBeanFactoryInitialization()方法进入，会预先初始化单例的bean，ServiceBean初始化过程会设置beanName,设置容器applicationContext,回调InitializingBean的afterPropertiesSet。
3. 在**afterPropertiesSet方法**里设置Provider、Application、Module、Registries、Monitor、Protocols、Path(beanName)，再由isDelay判断是否立即export。
4. finishRefresh()方法进入，publishEvent(..)当完成ApplicationContext初始化后，广播类ApplicationEventMulticaster广播ContextRefreshedEvent事件，以保证对应的监听器可以做进一步的逻辑处理，之前一步实例化的ServiceBean注册了这个事件，**触发onApplicationEvent并调用export()暴露服务**。
5. 获取前面ProviderConfig的export属性，判断是否暴露，不暴露的话直接返回。获取前面ProviderConfig的delay属性,如果需要delay，则delay几秒在调用doExport.否则，立即调用doExport。如同spring命名风格，真正做事的是doxxx方法，所以进入doExport()。
6. 将exported设为true，防止重复导出这个服务。判断interfaceName为空则抛出异常。
7. checkDefault()方法，appendProperties(provider)去读取xml配置(dubbo.properties)(System.getProperty)设置到provider(ProviderConfig)属性中去。




赋值application，module，registries，monitor，protocols。判断是否是GenericService还是自定义接口拿到interfaceClass。
7. 







# Consumer



1.服务提供端:ServiceConfig->ProxyFactory(JavassistProxyFactory或者JdkProxyFactory)->Invoker(AbstractProxyInvoker的实例)->filter(exception，monitor等)->RegistryProtocol(Dubbo,Hessian,Injvm,Rmi,WebService等)->exporter->server->transporter->serializtion->codec(telnet，transport，exchange)
2.消费调用端:jdk dynamic proxy->cluster->directory->registry->router->loadbalance->filter(monitor等)->invoker->client->transporter->codec








先在导出服务的service的bean配置中注入
ProviderConfig(threadPool=cached),
ApplicationConfig(dubbo.application.environment=test/develop/product, dubbo.application.provider.name=linglongta.test.provider, dubbo.monitor.isNeed=notNeed),
MonitorConfig(dubbo.monitor.address.port=10.168.163.241:7070),
RegistryConfig(dubbo.registry.address=10.168.163.241,dubbo.registry.port=2811,dubbo.registry.protocol=zookeeper),
ProtocolConfig(dubbo.application.protocol=dubbo,dubbo.protocol.port=-1).
在每个接口中，设置
interface,
group(dubbo.provider.group=testLingLongTaGroup),
application(前面的ApplicationConfig),
protocol(前面的ProtocolConfig),
registry(前面的RegistryConfig),
ref(注入的service实现),
timeout(dubbo.provider.timeout=300000).


