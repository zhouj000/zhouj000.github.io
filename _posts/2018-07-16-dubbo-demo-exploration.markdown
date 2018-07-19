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

1. ClassPathXmlApplicationContext设置xml地址。调用AbstractApplicationContext的refresh()方法。
2.  refresh的finishBeanFactoryInitialization()方法进入，会预先初始化单例的bean，ServiceBean初始化过程会设置beanName,设置容器applicationContext,回调InitializingBean的afterPropertiesSet。
![ServiceBean](/img/in-post/2018/ServiceBean.png)
3.  在**afterPropertiesSet方法**里设置Provider、Application、Module、Registries、Monitor、Protocols、Path(beanName)，再由isDelay判断是否立即export。
4.  refresh的finishRefresh()方法进入，publishEvent(..)当完成ApplicationContext初始化后，广播类ApplicationEventMulticaster广播ContextRefreshedEvent事件，以保证对应的监听器可以做进一步的逻辑处理，之前一步实例化的ServiceBean注册了这个事件，**触发onApplicationEvent并调用export()暴露服务**。
5.  获取前面ProviderConfig的export属性，判断是否暴露，不暴露的话直接返回。获取前面ProviderConfig的delay属性,如果需要delay，则delay几秒在调用doExport.否则，立即调用doExport。如同spring命名风格，真正做事的是doxxx方法，所以进入doExport()。
```java
protected synchronized void doExport() {
	if (unexported) {
		throw new IllegalStateException("Already unexported!");
	}
	if (exported) {
		return;
	}
	exported = true;
	if (interfaceName == null || interfaceName.length() == 0) {
		throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
	}
	checkDefault();
	if (provider != null) {
		if (application == null) {
			application = provider.getApplication();
		}
		if (module == null) {
			module = provider.getModule();
		}
		if (registries == null) {
			registries = provider.getRegistries();
		}
		if (monitor == null) {
			monitor = provider.getMonitor();
		}
		if (protocols == null) {
			protocols = provider.getProtocols();
		}
	}
	if (module != null) {
		if (registries == null) {
			registries = module.getRegistries();
		}
		if (monitor == null) {
			monitor = module.getMonitor();
		}
	}
	if (application != null) {
		if (registries == null) {
			registries = application.getRegistries();
		}
		if (monitor == null) {
			monitor = application.getMonitor();
		}
	}
	if (ref instanceof GenericService) {
		interfaceClass = GenericService.class;
		if (StringUtils.isEmpty(generic)) {
			generic = Boolean.TRUE.toString();
		}
	} else {
		try {
			interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
					.getContextClassLoader());
		} catch (ClassNotFoundException e) {
			throw new IllegalStateException(e.getMessage(), e);
		}
		checkInterfaceAndMethods(interfaceClass, methods);
		checkRef();
		generic = Boolean.FALSE.toString();
	}
	if (local != null) {
		if ("true".equals(local)) {
			local = interfaceName + "Local";
		}
		Class<?> localClass;
		try {
			localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
		} catch (ClassNotFoundException e) {
			throw new IllegalStateException(e.getMessage(), e);
		}
		if (!interfaceClass.isAssignableFrom(localClass)) {
			throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
		}
	}
	if (stub != null) {
		if ("true".equals(stub)) {
			stub = interfaceName + "Stub";
		}
		Class<?> stubClass;
		try {
			stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
		} catch (ClassNotFoundException e) {
			throw new IllegalStateException(e.getMessage(), e);
		}
		if (!interfaceClass.isAssignableFrom(stubClass)) {
			throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
		}
	}
	checkApplication();
	checkRegistry();
	checkProtocol();
	appendProperties(this);
	checkStubAndMock(interfaceClass);
	if (path == null || path.length() == 0) {
		path = interfaceName;
	}
	doExportUrls();
	ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
	ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```
6. 将exported设为true，防止重复导出这个服务。判断interfaceName为空则抛出异常。
7. checkDefault()方法，appendProperties(provider)去读取xml配置(dubbo.properties)(System.getProperty)设置到provider(ProviderConfig)属性中去。
8.  判断接口的实现类ref是否是GenericService类型，代表泛化调用，设置interfaceClass为GenericService类型。否则，通过反射设置interfaceClass为调用dubbo的interface接口类型，然后checkInterfaceAndMethods检查配置的MethodConfig中的方法是否都在该interface中存在，checkRef检查ref对象是否实现了interface接口。
9. (local = interfaceName + "Local" 与 stub = interfaceName + "Stub" 反射本地类/桩？)
10. 向后兼容(for backward compatibility)ApplicationConfig、RegistryConfig、ProtocolConfig。appendProperties再设置属性到ServiceConfig，checkStubAndMock按字面意思，检查stub和mock类。
11. 调用doExportUrls()导出URL。
```java
private void doExportUrls() {
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```
12. loadRegistries遍历所有的RegistryConfig，这里只有zookeeper，遍历时，获取注册中心的ip地址列表，将path、protocol、dubbo版本、时间戳等放到map中parseURLs根据ip地址的个数转成对应个数的URL字串的列表。将protocol设为registry。
13. 遍历不同的ProtocolConfig(dubbo，rmi，http，hessian，injvm等)，当前为dubbo，执行doExportUrlsFor1Protocol(protocolConfig, registryURLs)。
14. 建立map，放入各种参数(side, bind.ip, application, methods, qos.port, dubbo, interface, bind.port, anyhost等)。
15. findConfigedHosts()会从registryURLs地址列表遍历，将可以socket连接url设置为hostToBind，然后先从protocolConfig取默认ip，没有则返回hostToBind为host，获取port，创建URL，从URL获取scope，如果scope是none则不导出。
`dubbo://xxx.xxx.xxx.xx:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=xxx.xxx.xxx.xx&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=8020&qos.port=22222&side=provider&timestamp=1532010842960`
```java
String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
Integer port = this.findConfigedPorts(protocolConfig, name, map);
URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
		.hasExtension(url.getProtocol())) {
	url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
			.getExtension(url.getProtocol()).getConfigurator(url).configure(url);
}

String scope = url.getParameter(Constants.SCOPE_KEY);
// don't export when none is configured
if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

	// export to local if the config is not remote (export to remote only when config is remote)
	if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
		exportLocal(url);
	}
	// export to remote if the config is not local (export to local only when config is local)
	if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
		if (logger.isInfoEnabled()) {
			logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
		}
		if (registryURLs != null && registryURLs.size() > 0) {
			for (URL registryURL : registryURLs) {
				url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
				URL monitorUrl = loadMonitor(registryURL);
				if (monitorUrl != null) {
					url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
				}
				if (logger.isInfoEnabled()) {
					logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
				}
				Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
				DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

				Exporter<?> exporter = protocol.export(wrapperInvoker);
				exporters.add(exporter);
			}
		} else {
			Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
			DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

			Exporter<?> exporter = protocol.export(wrapperInvoker);
			exporters.add(exporter);
		}
	}
}
this.urls.add(url);
```
16. 判断scope不是remote，这里exportLocal(url)。
`injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=xxx.xxx.xxx.xx&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=8020&qos.port=22222&side=provider&timestamp=1532010842960`
```java
if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
	URL local = URL.valueOf(url.toFullString())
			.setProtocol(Constants.LOCAL_PROTOCOL)
			.setHost(LOCALHOST)
			.setPort(0);
	ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));
	Exporter<?> exporter = protocol.export(
			proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
	exporters.add(exporter);
	logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
}
```







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


