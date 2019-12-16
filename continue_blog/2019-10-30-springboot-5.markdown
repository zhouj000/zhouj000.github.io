---
layout:     post
title:      "SpringBoot(五) 健康检查"
date:       2019-10-12
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - springBoot
--- 

[SpringBoot(一) 启动与自动配置](https://zhouj000.github.io/2018/10/05/springboot-1/)  
[SpringBoot(二) starter与servlet容器](https://zhouj000.github.io/2018/10/10/springboot-2/)  
[SpringBoot(三) Environment](https://zhouj000.github.io/2019/10/08/springboot-3/)  
[SpringBoot(四) 集成apollo遇到的事儿](https://zhouj000.github.io/2019/10/16/springboot-4/)  




# 配置

springBoot提供的健康检查只需要引入依赖即可：
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
`spring-boot-starter-actuator`其实只是个空项目，统一管理的引入了：
```
// 自动配置
+ spring-boot-actuator-autoconfigure
	// actuator
	+ spring-boot-actuator
// 提供了记录度量指标类
// SpringBoot 2.X使用mircrometer进行统计数据和发布数据到监控系统，1.X是使用了dropwizard-metrics
+ micrometer-core
```
启动项目，可以看到打印了`Exposing 2 endpoint(s) beneath base path '/actuator'`，我们访问这个路径看到打印了：
```
{
	"_links": {
		"self": {
			"href": "http://127.0.0.1:8080/actuator",
			"templated": false
		},
		"health-component": {
			"href": "http://127.0.0.1:8080/actuator/health/{component}",
			"templated": true
		},
		"health-component-instance": {
			"href": "http://127.0.0.1:8080/actuator/health/{component}/{instance}",
			"templated": true
		},
		"health": {
			"href": "http://127.0.0.1:8080/actuator/health",
			"templated": false
		},
		"info": {
			"href": "http://127.0.0.1:8080/actuator/info",
			"templated": false
		}
	}
}
```
其实在SpringBoot2.X以前，有许多监控端点是打开的，比如env、beans等。但是2.X以后有以下规则：  
1、要通过actuator暴露端点，必须同时是**启用(enabled)**的和**暴露(exposed)**的  
2、所有**除了**/health和/info的端点，默认都是不暴露的  
3、所有**除了**/shutdown的端点，默认都是启用的(enabled)  
4、生产环境由于安全性的问题，注意不要暴露敏感端点(如果使用Spring Security，则可以使用Spring Security的内容协商策略保护端点)

通过`application.properties`或`application.ymal`配置可以打开这些监控端点：
```
// 启用指定监控端点，只有shutdown是默认未启用的
management.endpoint.shutdown.enabled=true

// 只暴露/排除指定监控端点，可以使用*表示全部
// 全部暴露，除了env端点
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env

// 修改访问根路径
management.endpoints.web.base-path=/monitor

// JMX配置
management.endpoints.jmx.exposure.include=
management.endpoints.jmx.exposure.exclude=
```
这样启动后就会看到`Exposing 12 endpoint(s) beneath base path '/monitor'`

|   类型   |     名称    | 作用 										 |
| :------: | :----------:| :-------------------------------------------- |
| Sensor   | auditevents | 显示当前应用程序的审计事件信息                |
|          | beans       | 显示应用Spring Beans的完整列表                |
|          | caches      | 显示可用缓存信息								 |
|          | conditions  | 显示自动装配类的状态及及应用信息              | 
|          | configprops | 显示所有 @ConfigurationProperties 列表        |
|          | env		 | 显示 ConfigurableEnvironment 中的属性         |
|          | info	     | 提供当前SpringBoot应用的任意信息，可以通过Environment或application.properties等形式提供以info.为前缀的任何配置项，然后info这个endpoint就会将这些配置项的值作为信息的一部分展示出来 |
|          | health      | 显示应用的健康信息(未认证只显示status，认证显示全部信息详情) |
|          | metrics 	 | 当前SprinBoot应用的metrics信息				 |
|          | httptrace	 | 显示HTTP跟踪信息(默认显示最后100个HTTP请求 - 响应交换) |
|          | mappings	 | 如果是基于SpringMVC的Web应用，显示所有 @RequestMapping 路径集列表 |
|          | scheduledtasks | 显示应用程序中的计划任务                   |
|          | sessions    | 允许从Spring会话支持的会话存储中检索和删除用户会话 |
| Actuator | shutdown	 | 用于关闭当前SpringBoot应用的endpoint 		 |
|          | dump		 | 用于执行线程的dump操作，heapdump和threaddump  |

# autoconfigure包

首先打开`spring-boot-actuator-autoconfigure`包，查看它的`spring.factories`，可以看到配置了许多配置类，都是其`org.springframework.boot.actuate.autoconfigure`路径下的文件

总体而言，autoconfigure下可以分为3类：  
1、自动配置XX端点
2、endpoint包
3、metrics包

其中自动配置按照分类，定义了各种**XXXEndpointAutoConfiguration**，比如rabbitmq、cache、bean等。其中会根据某些类(依赖)是否存在来自动创建**XXXEndpoint**，这些类使用`@Endpoint`注解类，`@ReadOperation、@WriteOperation或@DeleteOperation`注解方法(自动地通过JMX公开，在web应用程序中会通过HTTP公开)。也可以使用`@JmxEndpoint`或`@WebEndpoint`来编写特定于技术的端点，这些端点仅限于各自的技术，分别通过JMX公开和  HTTP公开。此外，可以使用`@EndpointWebExtension`和`@EndpointJmxExtension`编写特定于技术的扩展，这些注解允许你提供特定于技术的操作，以增强现有的端点。最后，如果需要访问特定于web框架的功能，可以实现Servlet或Spring的@Controller和@RestController端点，代价是它们在JMX上不可用，或者在使用不同的web框架时不可用

另一种是创建XXXHealthIndicator，这些类继承了AbstractHealthIndicator抽象类，实现了doHealthCheck方法

暂时先不关注reactive

与spring.factories同级下还有spring-autoconfigure-metadata.properties和spring-configuration-metadata.json配置文件，这2个只是格式不同，都是通过XXXProperties为部分XXXEndpointAutoConfiguration提供参数的
   

对于其中**endpoint**下主要有3部分组成：  
1、判断是否启用Endpoint：OnEnabledEndpointCondition#getMatchOutcome  
2、JMX导出：创建bean(JmxEndpointDiscoverer、JmxEndpointExporter、ExposeExcludePropertyEndpointFilter)  
3、WEB导出：按依赖分别创建**Jersey、Reactive、Spring MVC**3种定义的XXXEndpointHandlerMapping。对SERVLET类型的ServletEndpointManagementContextConfiguration创建ServletEndpointRegistrar；对WEB项目的WebEndpointAutoConfiguration创建WebEndpointDiscoverer、ControllerEndpointDiscoverer、PathMappedEndpoints和ExposeExcludePropertyEndpointFilter

回过来看一下上面配置是怎么生效的：
```
// AbstractEndpointCondition 判断指定配置
protected ConditionOutcome getEnablementOutcome(ConditionContext context, AnnotatedTypeMetadata metadata,
			Class<? extends Annotation> annotationClass) {
	Environment environment = context.getEnvironment();
	AnnotationAttributes attributes = getEndpointAttributes(annotationClass, context, metadata);
	EndpointId id = EndpointId.of(environment, attributes.getString("id"));
	String key = "management.endpoint." + id.toLowerCaseString() + ".enabled";
	Boolean userDefinedEnabled = environment.getProperty(key, Boolean.class);
	if (userDefinedEnabled != null) {
		return new ConditionOutcome(userDefinedEnabled, ConditionMessage.forCondition(annotationClass)
				.because("found property " + key + " with value " + userDefinedEnabled));
	}
	Boolean userDefinedDefault = isEnabledByDefault(environment);
	if (userDefinedDefault != null) {
		return new ConditionOutcome(userDefinedDefault, ConditionMessage.forCondition(annotationClass).because(
				"no property " + key + " found so using user defined default from " + ENABLED_BY_DEFAULT_KEY));
	}
	boolean endpointDefault = attributes.getBoolean("enableByDefault");
	return new ConditionOutcome(endpointDefault, ConditionMessage.forCondition(annotationClass)
			.because("no property " + key + " found so using endpoint default"));
}


// OnAvailableEndpointCondition
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
	ConditionOutcome enablementOutcome = getEnablementOutcome(context, metadata,
			ConditionalOnAvailableEndpoint.class);
	if (!enablementOutcome.isMatch()) {
		return enablementOutcome;
	}
	ConditionMessage message = enablementOutcome.getConditionMessage();
	Environment environment = context.getEnvironment();
	if (CloudPlatform.CLOUD_FOUNDRY.isActive(environment)) {
		return new ConditionOutcome(true, message.andCondition(ConditionalOnAvailableEndpoint.class)
				.because("application is running on Cloud Foundry"));
	}
	AnnotationAttributes attributes = getEndpointAttributes(ConditionalOnAvailableEndpoint.class, context,
			metadata);
	EndpointId id = EndpointId.of(environment, attributes.getString("id"));
	Set<ExposureInformation> exposureInformations = getExposureInformation(environment);
	for (ExposureInformation exposureInformation : exposureInformations) {
		// 判断是否导出
		if (exposureInformation.isExposed(id)) {
			return new ConditionOutcome(true,
					message.andCondition(ConditionalOnAvailableEndpoint.class)
							.because("marked as exposed by a 'management.endpoints."
									+ exposureInformation.getPrefix() + ".exposure' property"));
		}
	}
	return new ConditionOutcome(false, message.andCondition(ConditionalOnAvailableEndpoint.class)
			.because("no 'management.endpoints' property marked it as exposed"));
}

boolean isExposed(EndpointId endpointId) {
	String id = endpointId.toLowerCaseString();
	// 这里有我们配的env，所以env在这里会返回false
	if (!this.exclude.isEmpty()) {
		if (this.exclude.contains("*") || this.exclude.contains(id)) {
			return false;
		}
	}
	// 这里有我们配置的*，因此跳过
	if (this.include.isEmpty()) {
		if (this.exposeDefaults.contains("*") || this.exposeDefaults.contains(id)) {
			return true;
		}
	}
	// 由于是*，返回true
	return this.include.contains("*") || this.include.contains(id);
}
```
这样在**ConfigurationClassParser类**中不会因为conditionEvaluator.shouldSkip跳出，最后会执行doProcessConfigurationClass方法


最后看metrics，主要是做绑定数据到MeterRegistry

// 待



# 例子



https://segmentfault.com/a/1190000015309478?utm_source=tag-newest

























## 接收输入
端点上的操作通过它们的参数接收输入，当通过web公开时，这些参数的值取自URL的查询参数和JSON请求体，当通过JMX公开时，参数被映射到MBean操作的参数，默认情况下需要参数，可以使用@org.springframework.lang.Nullable对它们进行注解，从而使它们成为可选的

### 输入类型转换
如果需要，传递给端点操作方法的参数将自动转换为所需的类型，在调用操作方法之前，使用ApplicationConversionService实例将通过JMX或HTTP请求接收的输入转换为所需的类型


# 源码解析

## HealthIndicator

SpringBoot通过HealthIndicatorAutoConfiguration提供了一些常见服务的监控检查支持，比如：DataSourceHealthIndicator、RedisHealthIndicator、SolrHealthIndicator等
![]()
除了SpringBoot提供的检查，还可以通过提供HealthIndicator的实现，注册到IOC容器，让SpringBoot自动发现并使用它们

通常我们不直接实现HealthIndicator接口，而是继承AbstractHealthIndicator，然后只需要实现doHealthCheck方法就可以了。例如我们实现一个dubbo服务是否健康启用的检查，利用dubbo提供的EchoService直接检查dubbo的健康状态，如果没有异常就可以认为服务是健康状态的
```java
public class DubboHealthIndicator extends AbstractHealthIndicator {

    private final ReferenceBean bean;

    public DubboHealthIndicator(ReferenceBean bean) {
        this.bean = bean;
    }

    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        builder.withDetail("interface", bean.getObjectType());
        final EchoService service = (EchoService) bean.getObject();
        service.$echo("ok");
        builder.up();
    }
}
```
实现了HealthIndicator后，就需要注册在IOC中了，我们可以为所有dubbo服务的ReferenceBean都注册了，这样就不必为每个服务单独注册了，还可以做成starter自动配置(写入META-INF/spring.factories)，这样就自动对当前应用引用的所有dubbo服务进行健康检查了
```
@Configuration
@ConditionalOnClass(name = { "com.alibaba.dubbo.rpc.Exporter" })
public class DubboHealthIndicatorConfiguration {

    @Autowired
    HealthAggregator healthAggregator;

    @Autowired(required = false)
    Map<String, ReferenceBean> references;

    @Bean
    public HealthIndicator dubboHealthIndicator() {
        Map<String, HealthIndicator> indicators = new HashMap<>();
        for (String key : references.keySet()) {
            final ReferenceBean bean = references.get(key);
            indicators.put(key.startsWith("&") ? key.replaceFirst("&", "")
                    : key, new DubboHealthIndicator(bean));
        }
        return new CompositeHealthIndicator(healthAggregator, indicators);
    }
}
```
其中对于references，Spring框架支持依赖注入Key的类型为String的Map，遇到这种类型的Map声明，Spring将扫描容器中所有类型为T(这里为ReferenceBean)的bean，然后以该bean的name作为Map的Key，以bean实例作为对应的Value，从而构建一个Map并注入到依赖处



## endpoints

虽然我们定义了HealthIndicator，但它只是在应用内部，需要将它暴露出去访问才能实现出它的功效，这就需要将endpoints暴露给外部监控者，SpringBoot会议JmxEndpointExporter将所有Endpoint实例以JMX MBean的形式暴露，默认情况下这些Endpoint对应的JMX MBean会放在`org.springframework.boot`命名空间下面，不过可以通过`endpoints.jmx.domain`配置项进行更改，比如 `endpoints.jmx.domain=com.test.management`

自定义一个Endpoint，然后将这个Endpoint注册到IOC容器内就可以扩展actuator的功能了
```
@Endpoint(id = "hello")
public class HelloEndpoint {

    public HelloEndpoint() {
    }

    @ReadOperation
    public String hello() {
        return "hello world";
    }
}


@Bean
public HelloEndpoint helloEndpoint(){
	return new HelloEndpoint();
}
```
这样就可以访问`http://127.0.0.1:8080/actuator/hello`获取这个Endpoint的监控输出结果了

JMX和HTTP是endpoints对外开放访问最常用的方式，鉴于Java的序列化漏洞以及JMX的远程访问防火墙问题，因此建议用HTTP并配合安全防护将SpringBoot应用的endpoints开放给外部监控者使用



https://segmentfault.com/a/1190000015309478?utm_source=tag-newest
https://blog.csdn.net/Message_lx/article/details/89524795















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




## AbstractHealthIndicator




endpoints
JmxEndpointExporter

MetricsAutoConfiguration



















SOFABoot(   search: Readiness Check) 

https://blog.csdn.net/beyondself_77/article/details/80846270
https://blog.csdn.net/weixin_33912638/article/details/87978436


