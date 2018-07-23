---
layout:     post
title:      "Dubbo(一):从dubbo-demo初探dubbo源码"
date:       2018-07-16 23:00
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - java
    - rpc
--- 

<font id="last-updated">最后更新于：2018-07-23</font>

# 准备工作

源码从[github](https://github.com/apache/incubator-dubbo)上clone，本文使用的版本是2.6.1

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

## 注册/注销服务时序图
![dubbo-export](/img/in-post/2018/7/dubbo-export.jpg)	


### ServiceBean

![ServiceBean](/img/in-post/2018/7/serviceBean.png)

### URL

com.alibaba.dubbo.common.URL: 所有扩展点参数都包含URL参数，URL作为上下文信息贯穿整个扩展点设计体系。  
属性有protocol、host、port、path、parameters、string等。

### Protocol

![protocol](/img/in-post/2018/7/protocol.png)

主要方法： Exporter<T> export(Invoker&lt;T> invoker) 与 Invoker<T> refer(Class&lt;T> type, URL url)

### Invoker与Exporter

主要就是: 最外面用xxWrapper包xxExporter再包xxInvoker，Invoker最里面是个Wrapper(proxy:ref)

Invoker执行过程分成三种类型：
![invoker](/img/in-post/2018/7/invoker.png)

Exporter负责维护invoker的生命周期，只有Invoker&lt;T> getInvoker()方法与void unexport()方法。



## DEBUG

首先修改dubbo-demo-provider.xml配置文件，将dubbo:registry设置为`zookeeper://127.0.0.1:2181`  
启动Provider的main方法。

(部分相关代码在步骤下贴出)

1 . ClassPathXmlApplicationContext设置xml地址。调用AbstractApplicationContext的refresh()方法。

2 . obtainFreshBeanFactory() -> refreshBeanFactory() -> loadBeanDefinitions(DefaultListableBeanFactory) ..-> registerBeanDefinitions(Document, Resource) -> parseBeanDefinitions(Element, BeanDefinitionParserDelegate)时判断ele不是DefaultNamespace，&lt;dubbo:>下执行parseCustomElement(Element)，获取dubbo的namespaceUrl，取到DubboNamespaceHandler执行**DubboBeanDefinitionParser.parse**方法生成BeanDefinition。分别有(ApplicationConfig，RegistryConfig, ProtocolConfig, ServiceBean)。

3 . refresh的finishBeanFactoryInitialization()方法进入，会预先初始化单例的bean，ServiceBean初始化过程会设置beanName,设置容器applicationContext,回调InitializingBean的afterPropertiesSet。

4 . 在**afterPropertiesSet方法**里设置Provider、Application、Module、Registries、Monitor、Protocols、Path(beanName)，再由isDelay判断是否立即export。

5 . refresh的finishRefresh()方法进入，publishEvent(..)当完成ApplicationContext初始化后，广播类ApplicationEventMulticaster广播ContextRefreshedEvent事件，以保证对应的监听器可以做进一步的逻辑处理，之前一步实例化的ServiceBean注册了这个事件，**触发onApplicationEvent并调用export()暴露服务**。

***

6 . 获取ProviderConfig的export属性，判断是否暴露，不暴露的话直接返回。获取ProviderConfig的delay属性,如果需要则delay几秒再调用doExport。否则立即调用doExport。如同spring命名风格，真正做事的是doxxx方法，所以进入doExport()。
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
7 . 将exported设为true，防止重复导出这个服务。判断interfaceName为空则抛出异常。

8 . checkDefault()方法，appendProperties(provider)去读取xml配置(这里是dubbo.properties)设置到provider(ProviderConfig)属性中去。

9 . 判断接口的实现类ref是否是GenericService类型，代表泛化调用，设置interfaceClass为GenericService类型。否则，通过反射设置interfaceClass为调用dubbo的interface接口类型，然后checkInterfaceAndMethods检查配置的MethodConfig中的方法是否都在该interface中存在，checkRef检查ref对象是否实现了interface接口。

10 . local = interfaceName + "Local" 与 stub = interfaceName + "Stub"，反射判断是否实现interfaceClass接口(没进入)

11 . 向后兼容(for backward compatibility)ApplicationConfig、RegistryConfig、ProtocolConfig。appendProperties再设置属性到ServiceConfig，checkStubAndMock按字面意思，检查local，stub和mock。

12 . 调用doExportUrls()导出URL，进入方法。

13 . loadRegistries遍历所有的RegistryConfig，这里只配了zookeeper。遍历registries时，获取注册中心的ip地址列表，将path、protocol、dubbo版本、时间戳等放到map中。parseURLs根据ip和map转成对应个数的URL字串的列表。将URL协议头从zookeeper://设为**registry://**。返回registryURLs`registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=4612&qos.port=22222&registry=zookeeper&timestamp=1532191131293`。
```java
private void doExportUrls() {
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```
14 . 遍历不同的ProtocolConfig(dubbo，rmi，http，hessian，injvm等)，现在只配置了dubbo，执行doExportUrlsFor1Protocol(ProtocolConfig, List&lt;URL>)方法。

15 . 建立map，放入各种参数(side, bind.ip, application, methods, qos.port, dubbo, interface, bind.port, anyhost等)。

16 . 获取host，findConfigedHosts(ProtocolConfig, List&lt;URL>, Map&lt;String, String>)会从registryURLs地址列表遍历，将可以socket成功连接的注册url设置为hostToBind，然后先从protocolConfig取默认ip，没有则返回hostToBind为host。获取port，从dubbo:protocol配置中获取。最后创建**URL对象**`dubbo://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=<registry-host>&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&
methods=sayHello&pid=8020&qos.port=22222&side=provider&timestamp=1532191131293`，从URL获取scope(null)，如果scope是none则不导出。
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
17 . 判断scope不是remote，执行exportLocal(URL)本地暴露。首先根据`dubbo://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?参数`生成本地URL`injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?参数`。
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
18 . 然后在ServiceClassHolder的ThreadLocal中放入ref引用的class(DemoServiceImpl)。

19 . 接着先从**ProxyFactory(@SPI("javassist"))**(StubProxyFactoryWrapper)获取Invoker，内部调用JavassistProxyFactory.getInvoker，生成Wrapper包装proxy，返回AbstractProxyInvoker。然后通过**Protocol(@SPI("dubbo"))**(ProtocolListenerWrapper去export，这里判断protocol(injvm)是否是registry，现在不是则返回ListenerExporterWrapper，里面通过ProtocolFilterWrapper同样判断protocol，不是则buildInvokerChain建立Filter链，最终通过调用**InjvmProtocol**去创建InjvmExporter返回。完成导出的Exporter放入exporters列表中。
```java
// JavassistProxyFactory
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

// ProtocolListenerWrapper 返回ListenerExporterWrapper(Exporter<T>, List<ExporterListener>)
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
		return protocol.export(invoker);
	}
	return new ListenerExporterWrapper<T>(protocol.export(invoker),
			Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
					.getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
}

// ProtocolFilterWrappe
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
		return protocol.export(invoker);
	}
	return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
}

// InjvmProtocol 返回InjvmExporter(Invoker<T>, String, Map<String, Exporter<?>>)
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```
```
包装成：
ListenerExportWrapper( InjvmExporter( AbstractProxyInvoker( ... ), key, exporterMap[key, Invoker] ), List<ExporterListener )
AbstractProxyInvoker内部有: proxy:DemoServiceImpl, type:interfaceClass, url: injvm://127.0.0.1/..., wrapper等。
且为InvokerChain链，拥有next的Invoker引用，内部为filter实现调用。
```
![protocol-filters](/img/in-post/2018/7/protocol-filters.png)
20 . export to remote if the config is not local，由于scope为null，所以继续。

21 . 遍历registryURLs所有地址列表，加载monitor，设置参数导出MonitorURL，如果有Monitor则给注册url设置monitor参数。

22 . 根据服务具体实现ref、实现接口interfaceClass、regitryUrl从代理工厂**ProxyFactory(@SPI("javassist"))**(StubProxyFactoryWrapper)获取代理Invoker，依旧默认使用javassist库做反射(URL.getParameter:defaultValue)，通过JavassistProxyFactory.getInvoker()获取Invoker。

23 . 包装成DelegateProviderMetaDataInvoker(Invoker&lt;T>, ServiceConfig)。执行protocol.export(Invoker&lt;T>)，依旧通过ProtocolListenerWrapper去export，这次判断protocol是registry，通过ProtocolFilterWrapper判断后进入RegistryProtocol.export(final Invoker&lt;T>)。
```
包装成： originInvoker
DelegoteProviderMetaDataInvoker( AbstractProxyInvoker( ..., registry://127.0.0.1:2181/.. ), serviceConfig:this )
```  
24 . 先导出invoker暴露服务，先按key(`dubbo://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?参数`)从本地map缓存中取，没有则生成Invoker，使用protocol.export导出，然后放入map中。这里使用DubboProtocol，先根据invoker的url生成cacheKey(com.alibaba.dubbo.demo.DemoService:20880)，再根据invoker、cacheKey、exporterMap创建一个DubboExporter按key放入**exporterMap**中，从URL中判断是否是代理支持的事件和是否是回调服务。这里都不是。然后调用openServer(URL)方法根据Url创建一个Server放入serverMap，key为(&lt;registry-host>:20880)。createServer，默认使用netty，dubbo协议。启动Server。最后optimizeSerialization给url加上参数。
```
包装成：
ExporterChangeableWrapper( ListenerExporterWrapper( DubboExporter( InvokerDelegete( originInvoker, url ), key, exporterMap[key, invoker] ), List<ExporterListener> ) )
InvokerDelegete内url: dubbo://<registry-host>:20880/...，originInvoker为前面包装后的Invoker。且为InvokerChain链，拥有next的Invoker引用，内部为filter实现调用。
```
```java
// DubboProtocol
createServer(URL url) ->
server = Exchangers.bind(url, requestHandler)

// Exchangers
return getExchanger(url).bind(url, handler);

// HeaderExchanger
return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
// startHeatbeatTimer()

// Transporters
// 有Grizzly、Mina、Netty4、Netty(走了这个)
return getTransporter().bind(url, handler);

// NettyTransporter
return new NettyServer(url, listener);

// 创建server监听，NettyHandler处理Channel的连接/关闭，读/写事件
NettyServer.doOpen()
// pipeline.addLast("handler", nettyHandler)
```
```
这里传给netty的listener是DubboProtocol内部定义的一个ExchangeHandler
里面的reply(ExchangeChannel channel, Object message)方法为主要执行方法。接收事件后会到这个方法内执行，或者调用父类super.received(channel, message)。
所以当netty接受到请求时，会先
Invoker<?> invoker = getInvoker(channel, inv)获取，方法里生成serviceKey，从exporterMap里获取exporter，再返回exporter.getInvoker()
最后执行invoker.invoke(Invocation)，一路跟上去，包括一个invokerchain的filter处理链，
最后就是执行JavassistProxyFactory里生成的AbstractProxyInvoker的doInvoke方法。
内部执行wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments)反射执行，proxy为DemoServiceImpl。
```
![openServer](/img/in-post/2018/7/openServer.png)
25 . 回到RegistryProtocol，再注册provider，从originInvoker获取registryUrl(`zookeeper://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?参数`)，Registry(ZookeeperRegistry)与registedProviderUrl(`dubbo://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?参数`)，放入ProviderConsumerRegTable的本地providerInvokers(ConcurrentHashMap)中，判断是否延迟暴露，非延迟则registry.register(registedProviderUrl)完成注册(ZookeeperRegistry.doRegister(url)，通过zkClient创建URL对应节点)**[^1]**(消费者在消费服务时会根据消费的接口名找到对应的zookeeper节点目录，对目录进行监听，接收推送)，将ProviderConsumerRegTable中取出ProviderWrapper设置reg为true。
![providerInvokers](/img/in-post/2018/7/providerInvokers.png)
```java
// RegistryProtocol
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
	//export invoker
	final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

	URL registryUrl = getRegistryUrl(originInvoker);

	//registry provider
	final Registry registry = getRegistry(originInvoker);
	final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

	//to judge to delay publish whether or not
	boolean register = registedProviderUrl.getParameter("register", true);

	ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

	if (register) {
		register(registryUrl, registedProviderUrl);
		ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
	}

	// Subscribe the override data
	// FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
	// 当提供者订阅时，它将影响场景：某个JVM公开服务并调用相同的服务。因为订阅是用服务的名称缓存的，所以它会导致订阅信息被覆盖。
	final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
	final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
	overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
	registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
	//Ensure that a new exporter instance is returned every time export
	return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
}
```
26 . 
去发布订阅时覆盖数据，按registedProviderUrl创建overrideSubscribeUrl(`provider://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?参数`)和OverrideListener，执行registry.subscribe(URL, NotifyListener)，通过FailbackRegistry去执行ZookeeperRegistry.doSubscribe发送一个订阅请求，在zk上创建/dubbo/com.alibaba.dubbo.demo.DemoService/configurators节点**[^2]**，添加子节点Listener，如果节点下有修改触发notify。
```java
List<URL> urls = new ArrayList<URL>();
for (String path : toCategoriesPath(url)) {
	ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
	if (listeners == null) {
		zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
		listeners = zkListeners.get(url);
	}
	ChildListener zkListener = listeners.get(listener);
	if (zkListener == null) {
		listeners.putIfAbsent(listener, new ChildListener() {
			public void childChanged(String parentPath, List<String> currentChilds) {
				ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
			}
		});
		zkListener = listeners.get(listener);
	}
	zkClient.create(path, false);
	List<String> children = zkClient.addChildListener(path, zkListener);
	if (children != null) {
		urls.addAll(toUrlsWithEmpty(url, path, children));
	}
}
notify(url, listener, urls);
```
27 . 然后notify(URL, NotifyListener, List&lt;URL>)，调用AbstractRegistry.notify，获取URL的category作为key，将urls放入map的value中。遍历map结果，本地的file文件保存version等属性(同步/异步线程池保存)。调用OverrideListener.notify(List&lt;URL>)，获取当前currentUrl(可能已被多次覆盖)与newUrl(此次配置)，如果不一致，doChangeLocalExport(originInvoker, newUrl)，重新导出invoker。

28 . 回到RegistryProtocol，组装成DestroyableExporter(Exporter&lt;T> exporter, Invoker&lt;T> originInvoker, URL subscribeUrl, URL registerUrl)返回，类实现了unexport方法。返回Exporter。

29 . 回到ServiceConfig，向exporters中加入导出完成的Exporter，此时就有2个了，一个是本地ListenerExportWrapper，一个是注册到远程ZK的Registryprotocol$ExporterChangeableWrapper，最后在ServiceConfig的urls加入 `dubbo://<registry-host>:20880/com.alibaba.dubbo.demo.DemoService?参数`。
![exporters](/img/in-post/2018/7/exporters.png)
30 . 创建ProviderModel(String serviceName, ServiceConfig metadata, Object serviceInstance)，初始化initProviderModel，放入ApplicationModel的providedServices(ConcurrentMap)中，key为com.alibaba.dubbo.demo.DemoService，value为ProviderModel。


#### zookeeper下node结构
```
^1:
/dubbo
	/com.alibaba.dubbo.demo.DemoService
		/providers[dubbo://xxx.xxx.xxx.xx:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=8376&side=provider&timestamp=1532179751055]	
		
^2:
/dubbo
	/com.alibaba.dubbo.demo.DemoService
		/providers
		/configurators[]


```



# Consumer

## 服务订阅/取消时序图
![dubbo-refer](/img/in-post/2018/7/dubbo-refer.jpg)	

## DEBUG

首先修改dubbo-demo-consumer.xml配置文件，将dubbo:registry设置为`zookeeper://127.0.0.1:2181`  
启动Consumer的main方法。

(部分相关代码在步骤下贴出)

1 . 同Provider一样，ClassPathXmlApplicationContext初始化，


jdk dynamic proxy -> cluster -> directory -> registry -> router -> loadbalance -> filter(monitor等) -> invoker -> client -> transporter -> codec















# 实现细节

## 初始化过程细节

#### 解析服务

基于 dubbo.jar 内的 META-INF/spring.handlers 配置，Spring 在遇到 dubbo 名称空间时，会回调 DubboNamespaceHandler。

所有 dubbo 的标签，都统一用 DubboBeanDefinitionParser 进行解析，基于一对一属性映射，将 XML 标签解析为 Bean 对象。

在 ServiceConfig.export() 或 ReferenceConfig.get() 初始化时，将 Bean 对象转换 URL 格式，所有 Bean 属性转成 URL 的参数。

然后将 URL 传给 [协议扩展点](http://dubbo.apache.org/#!/docs/dev/impls/protocol.md?lang=zh-cn)，基于扩展点的 [扩展点自适应机制](http://dubbo.apache.org/#!/docs/dev/SPI.md?lang=zh-cn)，根据 URL 的协议头，进行不同协议的服务暴露或引用。

#### 暴露服务

###### 1. 只暴露服务端口：

在没有注册中心，直接暴露提供者的情况下，ServiceConfig 解析出的 URL 的格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`。

基于扩展点自适应机制，通过 URL 的 **dubbo://** 协议头识别，直接调用 DubboProtocol的 export() 方法，打开服务端口。

###### 2. 向注册中心暴露服务：

在有注册中心，需要注册提供者地址的情况下，ServiceConfig 解析出的 URL 的格式为: `registry://registry-host/com.alibaba.dubbo.registry.RegistryService?export=URL.encode("dubbo://service-host/com.foo.FooService?version=1.0.0")`。

基于扩展点自适应机制，通过 URL 的 **registry://** 协议头识别，就会调用 RegistryProtocol 的 export() 方法，将 export 参数中的提供者 URL，先注册到注册中心。

再重新传给 Protocol 扩展点进行暴露： `dubbo://service-host/com.foo.FooService?version=1.0.0`，然后基于扩展点自适应机制，通过提供者 URL 的 **dubbo://** 协议头识别，就会调用 DubboProtocol 的 export() 方法，打开服务端口。

#### 引用服务

###### 1. 直连引用服务：

在没有注册中心，直连提供者的情况下，ReferenceConfig 解析出的 URL 的格式为：`dubbo://service-host/com.foo.FooService?version=1.0.0`。

基于扩展点自适应机制，通过 URL 的 **dubbo://** 协议头识别，直接调用 DubboProtocol 的 refer() 方法，返回提供者引用。

###### 2. 从注册中心发现引用服务：

在有注册中心，通过注册中心发现提供者地址的情况下，ReferenceConfig 解析出的 URL 的格式为：`registry://registry-host/com.alibaba.dubbo.registry.RegistryService?refer=URL.encode("consumer://consumer-host/com.foo.FooService?version=1.0.0")`。

基于扩展点自适应机制，通过 URL 的 **registry://** 协议头识别，就会调用 RegistryProtocol 的 refer() 方法，基于 refer 参数中的条件，查询提供者 URL，如： `dubbo://service-host/com.foo.FooService?version=1.0.0`。

基于扩展点自适应机制，通过提供者 URL 的 **dubbo://** 协议头识别，就会调用 DubboProtocol 的 refer() 方法，得到提供者引用。

然后 RegistryProtocol 将多个提供者引用，通过 Cluster 扩展点，伪装成单个提供者引用返回。

#### 拦截服务

基于扩展点自适应机制，所有的 Protocol 扩展点都会自动套上 Wrapper 类。

基于 ProtocolFilterWrapper 类，将所有 Filter 组装成链，在链的最后一节调用真实的引用。

基于 ProtocolListenerWrapper 类，将所有 InvokerListener 和 ExporterListener 组装集合，在暴露和引用前后，进行回调。

包括监控在内，所有附加功能，全部通过 Filter 拦截实现。

## 远程调用细节

#### 服务提供者暴露一个服务的详细过程

![rpc_export](/img/in-post/2018/7/dubbo_rpc_export.jpg)	

上图是服务提供者暴露服务的主过程：

首先 ServiceConfig 类拿到对外提供服务的实际类 ref(如：HelloWorldImpl),然后通过 ProxyFactory 类的 getInvoker 方法使用 ref 生成一个 AbstractProxyInvoker 实例，到这一步就完成具体服务到 Invoker 的转化。接下来就是 Invoker 转换到 Exporter 的过程。

Dubbo 处理服务暴露的关键就在 Invoker 转换到 Exporter 的过程，上图中的红色部分。下面我们以 Dubbo 和 RMI 这两种典型协议的实现来进行说明：

#### Dubbo 的实现

Dubbo 协议的 Invoker 转为 Exporter 发生在 DubboProtocol 类的 export 方法，它主要是打开 socket 侦听服务，并接收客户端发来的各种请求，通讯细节由 Dubbo 自己实现。

#### RMI 的实现

RMI 协议的 Invoker 转为 Exporter 发生在 RmiProtocol类的 export 方法，它通过 Spring 或 Dubbo 或 JDK 来实现 RMI 服务，通讯细节这一块由 JDK 底层来实现，这就省了不少工作量。

#### 服务消费者消费一个服务的详细过程

![rpc_refe](/img/in-post/2018/7/dubbo_rpc_refer.jpg)

上图是服务消费的主过程：

首先 ReferenceConfig 类的 init 方法调用 Protocol 的 refer 方法生成 Invoker 实例(如上图中的红色部分)，这是服务消费的关键。接下来把 Invoker 转换为客户端需要的接口(如：HelloWorld)。

关于每种协议如 RMI/Dubbo/Web service 等它们在调用 refer 方法生成 Invoker 实例的细节和上一章节所描述的类似。

## 远程通讯细节

#### 协议头约定

![dubbo_protocol_header](/img/in-post/2018/7/dubbo_protocol_header.jpg)

#### 线程派发模型

![dubbo-protocol](/img/in-post/2018/7/dubbo-protocol.jpg)
+ Dispather: all, direct, message, execution, connection
+ ThreadPool: fixed, cached

[官方文档: 实现细节](http://dubbo.apache.org/#!/docs/dev/implementation.md?lang=zh-cn)
