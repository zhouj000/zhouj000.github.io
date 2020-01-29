---
layout:     post
title:      "SpringBoot(六) 健康检查(下)"
date:       2020-02-01
author:     "ZhouJ000"
header-img: "img/in-post/2020/post-bg-2020-headbg.jpg"
catalog: true
tags:
    - springBoot
--- 

[SpringBoot(一) 启动与自动配置](https://zhouj000.github.io/2018/10/05/springboot-1/)  
[SpringBoot(二) starter与servlet容器](https://zhouj000.github.io/2018/10/10/springboot-2/)  
[SpringBoot(三) Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)  
[SpringBoot(四) 集成apollo遇到的事儿](https://zhouj000.github.io/2019/10/16/springboot-4/)  
[SpringBoot(五) 健康检查(上)](https://zhouj000.github.io/2020/01/28/springboot-5/)  




# 配置











## Endpoint

现在仔细看下autoconfigure第二部分-endpoint

WebEndpointAutoConfiguration主要创建WebEndpointDiscoverer、PathMappedEndpoints

WebMvcEndpointManagementContextConfiguration创建
WebMvcEndpointHandlerMapping 
















## EndpointDiscoverer

入口是由META-INF/spring.factories配置的ServletEndpointManagementContextConfiguration
```
@ManagementContextConfiguration
@ConditionalOnWebApplication(type = Type.SERVLET)
public class ServletEndpointManagementContextConfiguration {
	@Bean
	public ExposeExcludePropertyEndpointFilter<ExposableServletEndpoint> servletExposeExcludePropertyEndpointFilter(
			WebEndpointProperties properties) {
		WebEndpointProperties.Exposure exposure = properties.getExposure();
		return new ExposeExcludePropertyEndpointFilter<>(ExposableServletEndpoint.class, exposure.getInclude(),
				exposure.getExclude());
	}
	
	@Configuration
	@ConditionalOnClass(DispatcherServlet.class)
	public static class WebMvcServletEndpointManagementContextConfiguration {

		private final ApplicationContext context;

		public WebMvcServletEndpointManagementContextConfiguration(ApplicationContext context) {
			this.context = context;
		}

		@Bean
		public ServletEndpointRegistrar servletEndpointRegistrar(WebEndpointProperties properties,
				ServletEndpointsSupplier servletEndpointsSupplier) {
			DispatcherServletPath dispatcherServletPath = this.context.getBean(DispatcherServletPath.class);
			// basePath:"/actuator", servletEndpoints: ---->
			return new ServletEndpointRegistrar(dispatcherServletPath.getRelativePath(properties.getBasePath()),
					servletEndpointsSupplier.getEndpoints());
		}
	}
	// ....
}
```
于是就创建EndpointDiscoverer，调用getEndpoints方法，由于是第一次，因此需要createEndpointBeans，去创建所有@Endpoint的类
```
private Collection<EndpointBean> createEndpointBeans() {
	Map<EndpointId, EndpointBean> byId = new LinkedHashMap<>();
	String[] beanNames = BeanFactoryUtils.beanNamesForAnnotationIncludingAncestors(this.applicationContext,
			Endpoint.class);
	for (String beanName : beanNames) {
		if (!ScopedProxyUtils.isScopedTarget(beanName)) {
			EndpointBean endpointBean = createEndpointBean(beanName);
			EndpointBean previous = byId.putIfAbsent(endpointBean.getId(), endpointBean);
			Assert.state(previous == null, () -> "Found two endpoints with the id '" + endpointBean.getId() + "': '"
					+ endpointBean.getBeanName() + "' and '" + previous.getBeanName() + "'");
		}
	}
	return byId.values();
}
```
![]()

## JmxEndpointExporter

入口是JmxEndpointAutoConfiguration，
```
@Configuration
@AutoConfigureAfter(JmxAutoConfiguration.class)
@EnableConfigurationProperties(JmxEndpointProperties.class)
public class JmxEndpointAutoConfiguration {
	// ...

	@Bean
	@ConditionalOnSingleCandidate(MBeanServer.class)
	public JmxEndpointExporter jmxMBeanExporter(MBeanServer mBeanServer, Environment environment,
			ObjectProvider<ObjectMapper> objectMapper, JmxEndpointsSupplier jmxEndpointsSupplier) {
		String contextId = ObjectUtils.getIdentityHexString(this.applicationContext);
		EndpointObjectNameFactory objectNameFactory = new DefaultEndpointObjectNameFactory(this.properties, environment,
				mBeanServer, contextId);
		JmxOperationResponseMapper responseMapper = new JacksonJmxOperationResponseMapper(
				objectMapper.getIfAvailable());
		return new JmxEndpointExporter(mBeanServer, objectNameFactory, responseMapper,
				jmxEndpointsSupplier.getEndpoints());
	}
	
	// ...
}
```
创建了JmxEndpointExporter，其实现了InitializingBean
```
public class JmxEndpointExporter implements InitializingBean, DisposableBean, BeanClassLoaderAware
```
因此调用afterPropertiesSet，最后将EndpointMBean都注册到了MBeanServer(com.sun.jmx.mbeanserver)
```
@Override
public void afterPropertiesSet() {
	this.registered = register();
}

private Collection<ObjectName> register() {
	return this.endpoints.stream().map(this::register).collect(Collectors.toList());
}

private ObjectName register(ExposableJmxEndpoint endpoint) {
	Assert.notNull(endpoint, "Endpoint must not be null");
	try {
		ObjectName name = this.objectNameFactory.getObjectName(endpoint);
		EndpointMBean mbean = new EndpointMBean(this.responseMapper, this.classLoader, endpoint);
		this.mBeanServer.registerMBean(mbean, name);
		return name;
	} catch ...
	}
}
```


## doHealth()

JMX通过RMI连接器RMIConnectionImpl调用JmxMBeanServer，去执行EndpointMBean的invoke方法，比如传参是health，你们最终会找到HealthEndpoint，执行他的health方法：
```
@Endpoint(id = "health")
public class HealthEndpoint {
	private final HealthIndicator healthIndicator;
	// ...
	@ReadOperation
	public Health health() {
		return this.healthIndicator.health();
	}
	// ...
}

// CompositeHealthIndicator
@Override
public Health health() {
	Map<String, Health> healths = new LinkedHashMap<>();
	for (Map.Entry<String, HealthIndicator> entry : this.registry.getAll().entrySet()) {
		healths.put(entry.getKey(), entry.getValue().health());
	}
	return this.aggregator.aggregate(healths);
}

// AbstractHealthIndicator
@Override
public final Health health() {
	Health.Builder builder = new Health.Builder();
	try {
		doHealthCheck(builder);
	}
	catch (Exception ex) {
		if (this.logger.isWarnEnabled()) {
			String message = this.healthCheckFailedMessage.apply(ex);
			this.logger.warn(StringUtils.hasText(message) ? message : DEFAULT_MESSAGE, ex);
		}
		builder.down(ex);
	}
	return builder.build();
}
```
最后调用了DiskSpaceHealthIndicator的doHealthCheck来创建返回监控结果，和我们之前定义的DubboHealthIndicator类似

那么为什么最后是用DiskSpaceHealthIndicator呢？我们发现这是`this.registry.getAll().entrySet()`返回的唯一值，而this.registry是DefaultHealthIndicatorRegistry，从构造方法往上找，发现HealthIndicatorRegistryBeans中
```
public static HealthIndicatorRegistry get(ApplicationContext applicationContext) {
	Map<String, HealthIndicator> indicators = new LinkedHashMap<>();
	// 这里只有DiskSpaceHealthIndicator
	indicators.putAll(applicationContext.getBeansOfType(HealthIndicator.class));
	if (ClassUtils.isPresent("reactor.core.publisher.Flux", null)) {
		new ReactiveHealthIndicators().get(applicationContext).forEach(indicators::putIfAbsent);
	}
	HealthIndicatorRegistryFactory factory = new HealthIndicatorRegistryFactory();
	return factory.createHealthIndicatorRegistry(indicators);
}
```
因此在DefaultHealthIndicatorRegistry中传入的healthIndicators只有这个

























  	



AbstractWebMvcEndpointHandlerMapping#initHandlerMethods    《- afterPropertiesSet

DispatcherServlet#doDispatch -> AbstractHandlerMethodAdapter#handle ->
handleInternal -> RequestMappingHandlerAdapter#invokeHandlerMethod ->
ServletInvocableHandlerMethod#invokeAndHandle //->
InvocableHandlerMethod#invokeForRequest -> #doInvoke ->
AbstractWebMvcEndpointHandlerMapping.OperationHandler#handle ->
AbstractWebMvcEndpointHandlerMapping.ServletWebOperationAdapter#handle ->
AbstractDiscoveredOperation#invoke -> EnvironmentEndpoint#environment


https://blog.csdn.net/songzehao/article/details/84979847
BeanNameUrlHandlerMapping
DispatcherServlet#initHandlerMappings




# 源码解析

## HealthIndicator

SpringBoot通过HealthIndicatorAutoConfiguration提供了一些常见服务的监控检查支持，比如：DataSourceHealthIndicator、RedisHealthIndicator、SolrHealthIndicator等
![]()
除了SpringBoot提供的检查，还可以通过提供HealthIndicator的实现，注册到IOC容器，让SpringBoot自动发现并使用它们

## HealthIndicatorAutoConfiguration

创建
```
@Bean
@ConditionalOnMissingBean(HealthAggregator.class)
public OrderedHealthAggregator healthAggregator() {
	OrderedHealthAggregator healthAggregator = new OrderedHealthAggregator();
	if (this.properties.getOrder() != null) {
		healthAggregator.setStatusOrder(this.properties.getOrder());
	}
	return healthAggregator;
}
```

## AbstractHealthIndicator


endpoints
JmxEndpointExporter

MetricsAutoConfiguration













# SOFABoot

SOFABoot在SpringBoot的Liveness检查能力的基础上，增加了Readiness检查能力



Kubelet使用liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness探针将捕获到deadlock，重启处于该状态下的容器，使应用程序在存在bug的情况下依然能够继续运行下去（谁的程序还没几个bug呢）。

Kubelet使用readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当Pod中的容器都处于就绪状态时kubelet才会认定该Pod处于就绪状态。该信号的作用是控制哪些Pod应该作为service的后端。如果Pod处于非就绪状态，那么它们将会被从service的load balancer中移除


ReadinessCheckListener


SOFABoot(   search: Readiness Check) 

https://blog.csdn.net/beyondself_77/article/details/80846270
https://blog.csdn.net/weixin_33912638/article/details/87978436










