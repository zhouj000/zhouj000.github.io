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
2、所有**除了**`/health`和`/info`的端点，默认都是不暴露的  
3、所有**除了**`/shutdown`的端点，默认都是启用的(enabled)  
4、生产环境由于安全性的问题，注意不要暴露敏感端点(如果使用Spring Security，则可以使用Spring Security的内容协商策略保护端点)

通过`application.properties`或`application.ymal`配置可以打开这些监控端点：
```
#management.endpoints.enabled-by-default=true
// 启用指定监控端点，只有shutdown是默认未启用的
management.endpoint.shutdown.enabled=true

// 只暴露/排除指定监控端点，可以使用*表示全部
// 全部暴露，除了env端点
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env

// 一个链接所有端点的”discovery页面”被添加，默认情况下，”discovery页面”在/actuator上可用
// 修改访问根路径，使用配置自定义的管理上下文路径，”discovery页面”自动从/actuator移动到管理上下文的根
management.endpoints.web.base-path=/monitor
// 修改端口映射，改变端点路径，比如重新映射/monitor/health到/monitor/healthcheck
management.endpoints.web.path-mapping.health=healthcheck

// <name>端点缓存的生存时间设置为10秒
management.endpoint.<name>.cache.time-to-live=10s

// CORS跨源资源共享设置，指定授权的跨域请求
management.endpoints.web.cors.allowed-origins=http://example.com
management.endpoints.web.cors.allowed-methods=GET,POST
// 使用不同的HTTP端口来公开端点
management.server.port=8081
management.server.XXX


//激活所有的JMX方式请求
management.endpoints.jmx.exposure.include=*
#management.endpoints.jmx.exposure.exclude=
// 不暴露JMX端点
#endpoints.jmx.enabled=false
```
这样启动后就会看到`Exposing 12 endpoint(s) beneath base path '/monitor'`

|   类型   |     名称    | 作用 										 |
| :------: | :----------:| :-------------------------------------------- |
| Sensor   | auditevents | 显示当前应用程序的审计事件信息                |
|          | beans       | 显示应用Spring Beans的完整列表                |
|          | caches      | 显示可用缓存信息								 |
|          | conditions  | 显示自动装配类的状态及及应用信息              | 
|          | configprops | 显示所有`@ConfigurationProperties`列表        |
|          | env		 | 显示 ConfigurableEnvironment 中的属性         |
|          | info	     | 提供当前SpringBoot应用的任意信息，可以通过Environment或application.properties等形式提供以info.为前缀的任何配置项，然后info这个endpoint就会将这些配置项的值作为信息的一部分展示出来 |
|          | health      | 显示应用的健康信息(未认证只显示status，认证显示全部信息详情) |
|          | metrics 	 | 当前SprinBoot应用的metrics信息				 |
|          | httptrace	 | 显示HTTP跟踪信息(默认显示最后100个HTTP请求 - 响应交换) |
|          | mappings	 | 如果是基于SpringMVC的Web应用，显示所有`@RequestMapping`路径集列表 |
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

其中自动配置按照分类，定义了各种**XXXEndpointAutoConfiguration**，比如rabbitmq、cache、bean等。其中会根据某些类(依赖)是否存在来自动创建**XXXEndpoint**，这些类使用`@Endpoint`注解类，`@ReadOperation、@WriteOperation或@DeleteOperation`注解方法(自动地通过JMX公开，在web应用程序中会通过HTTP公开)。也可以使用`@JmxEndpoint`或`@WebEndpoint`来编写特定于技术的端点，这些端点仅限于各自的技术，分别通过JMX公开和HTTP公开。此外，可以使用`@EndpointWebExtension`和`@EndpointJmxExtension`编写特定于技术的扩展，这些注解允许你提供特定于技术的操作，以增强现有的端点。最后，如果需要访问特定于web框架的功能，可以实现Servlet或Spring的`@Controller`和`@RestController`端点，代价是它们在JMX上不可用，或者在使用不同的web框架时不可用

另一种是创建**XXXHealthIndicator**，这些类继承了**AbstractHealthIndicator**抽象类，实现了`doHealthCheck`方法

暂时先不关注reactive

与spring.factories同级下还有`spring-autoconfigure-metadata.properties`和`spring-configuration-metadata.json`配置文件，这2个只是格式不同，都是通过**XXXProperties**为部分XXXEndpointAutoConfiguration提供参数的
   

对于其中**endpoint**下主要有3部分组成：  
1、判断是否**启用**Endpoint：OnEnabledEndpointCondition#getMatchOutcome  
2、**JMX导出**：创建bean(JmxEndpointDiscoverer、JmxEndpointExporter、ExposeExcludePropertyEndpointFilter)  
3、**WEB导出**：按依赖分别创建**Jersey、Reactive、Spring MVC**3种定义的XXXEndpointHandlerMapping。对SERVLET类型的ServletEndpointManagementContextConfiguration创建ServletEndpointRegistrar；对WEB项目的WebEndpointAutoConfiguration创建WebEndpointDiscoverer、ControllerEndpointDiscoverer、PathMappedEndpoints和ExposeExcludePropertyEndpointFilter  
即如果你添加一个带`@Endpoint`注解的`@Bean`，那么任何带`@ReadOperation、@WriteOperation或@DeleteOperation`的方法都会自动地通过JMX公开，在web应用程序中，也会通过HTTP公开，可以使用Jersey、Spring MVC或Spring WebFlux通过HTTP公开端点

回过来看一下上面配置启用是怎么生效的：
```
// OnAvailableEndpointCondition
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
	// 1. 获取条件结果，enabled配置
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
		// 2. 判断是否导出，expose配置
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


// 1. AbstractEndpointCondition 判断指定配置，没有用默认
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

// 2. 查看expose
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



# 自定义

## 自定义端点

端点上的操作通过它们的参数接收输入，当通过web公开时，这些参数的值取自**URL的查询参数和JSON请求体**，当通过JMX公开时，参数被映射到**MBean**操作的参数，默认情况下需要参数，可以使用`@org.springframework.lang.Nullable`对它们进行注解，从而使它们成为可选的

如果需要，传递给端点操作方法的参数将**自动转换**为所需的类型，在调用操作方法之前，使用**ApplicationConversionService**实例将通过JMX或HTTP请求接收的输入转换为所需的类型

#### Web端点

在web公开的端点上，每个操作都会自动生成一个请求谓词，谓词的路径由端点的ID和web公开端点的基本路径决定，例如现在的`/monitor/info`。通过使用`@Selector`注解操作方法的一个或多个参数，可以进一步定制路径，这样的参数作为路径变量添加到路径谓词，当调用端点操作时，将变量的值传递给操作方法

谓词的HTTP方法由操作类型决定，如下所示：  
```
@ReadOperation     => GET
@WriteOperation    => POST
@DeleteOperation   => DELETE
```
谓词的生产子句可以由`@DeleteOperation、@ReadOperation和@WriteOperation`注解的**produces**属性确定，属性是可选的，如果不使用，则自动确定"生成"子句。如果操作方法produces返回void或Void，则生成子句为空，如果操作方法返回一个`org.springframework.core.io.Resource`，生成子句是`application/octet-stream`。对于**所有其他操作**，生成子句是`application/vnd.spring-boot.actuator.v2+json, application/json`

对于使用请求体的`@WriteOperation`(HTTP POST)，谓词的消费子句是`application/vnd.spring-boot.actuator.v2+json, application/json`，**对于所有其他操作，消费子句为空**

#### Servlet端点

Servlet可以通过实现一个带有`@ServletEndpoint`注解的类来作为端点公开，这个类也实现了`Supplier<EndpointServlet>`。Servlet端点提供了与Servlet容器更深层次的集成，但暴露了可移植性，它们用于将现有的Servlet公开为端点。对于新的端点，应该尽可能使用`@Endpoint`和`@WebEndpoint`注解

#### Controller端点

可以使用`@ControllerEndpoint`和`@RestControllerEndpoint`实现仅由Spring MVC或Spring WebFlux公开的端点，方法使用Spring MVC和Spring WebFlux(如`@RequestMapping`和`@GetMapping`)的标准注解进行映射，使用端点的ID作为路径的前缀。Controller端点提供了与Spring web框架的更深入的集成，但却牺牲了可移植性，尽可能使用`@Endpoint`和`@WebEndpoint`注解

## 健康信息

health端点公开的信息取决于management.endpoint.health.show-details属性，可以使用以下值之一配置：  
1、never：默认，不显示细节  
2、when-authorized：详细信息只显示给授权用户，可以使用`management.endpoint.health.roles`配置授权角色(当用户处于端点的一个或多个角色中时，就被认为是经过授权的，如果端点没有配置角色(默认)，则认为所有经过身份验证的用户都是经过授权的)  
3、always：详细信息显示给所有用户

健康信息是从**ApplicationContext**中定义的所有**HealthIndicator bean**中收集的，Spring Boot包括许多**自动配置的HealthIndicators**，并且你也可以自己写。默认情况下，最终的系统状态由**HealthAggregator**派生，它根据有序的状态列表从**每个HealthIndicator**排序状态。排序列表中的**第一个**状态被用作**总体**健康状态，如果**没有**HealthAggregator所知道的HealthIndicator状态返回，则使用**UNKNOWN**状态

比如Spring Boot提供了一些自动配置的HealthIndicators：  
1、**CassandraHealthIndicator**：检查Cassandra数据库是否已启动  
2、**DiskSpaceHealthIndicator**：检查低磁盘空间  
3、**DataSourceHealthIndicator**：检查能否获得到DataSource的连接  
4、**RedisHealthIndicator**：检查Redis服务是否已启动  
等等

#### 自定义HealthIndicators

要提供自定义的健康信息，可以注册实现**HealthIndicator**接口的Spring bean，并提供`health()`方法的实现并返回Health响应

给定HealthIndicator的标识符是没有HealthIndicator后缀的bean的名称，例如MyHealthIndicator的健康信息可以在my条目中获得

除了Spring Boot的预定义**Status类型**之外，Health还可以返回表示新系统状态的自定义Status，在这种情况下，还需要提供**HealthAggregator接口的自定义实现**，或者，默认实现是使用`management.health.status.order`配置属性:
```
// 一个带有代码FATAL的新Status
management.health.status.order=FATAL, DOWN, OUT_OF_SERVICE, UNKNOWN, UP
// 通过HTTP访问健康端点，可能还想注册自定义状态映射
management.health.status.http-mapping.FATAL=503
```
已有的默认状态：  
1、**DOWN**：SERVICE_UNAVAILABLE (503)  
1、**OUT_OF_SERVICE**：SERVICE_UNAVAILABLE (503)  
1、**UP**：默认情况下没有映射，所以http状态是200  
1、**UNKNOWN**：默认情况下没有映射，所以http状态是200

## 例子

直接实现HealthIndicator接口
```
@Component("my_health_indicator")
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        return Health.up().withDetail("code", "0000").withDetail("msg","SUCCESS").up().build();
    }
}
```
可以通过`http://127.0.0.1:8080/actuator/health`或直接`http://127.0.0.1:8080/actuator/health/my_health_indicator`查看

另一种方式就是继承AbstractHealthIndicator抽象类，这是通常的做法，只需要实现doHealthCheck方法，比如我们实现一个通用的dubbo依赖的健康检查，利用EchoService判断
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
```java
@Configuration
@ConditionalOnClass(name = { "com.alibaba.dubbo.rpc.Exporter" })
public class DubboHealthIndicatorConfiguration {

    @Autowired
    StatusAggregator healthAggregator;

    @Autowired(required = false)
    Map<String, ReferenceBean> references;

    @Bean
    public CompositeHealthContributor dubboHealthIndicator() {
        Map<String, HealthIndicator> indicators = new HashMap<>();
        for (String key : references.keySet()) {
            final ReferenceBean bean = references.get(key);
            indicators.put(key.startsWith("&") ? key.replaceFirst("&", "")
                    : key, new DubboHealthIndicator(bean));
        }
        return CompositeHealthContributor.fromMap(indicators);
    }

}
```
其中对于references，Spring框架支持**依赖注入Key的类型为String的Map**，遇到这种类型的Map声明，Spring将扫描容器中所有类型为T(这里为ReferenceBean)的bean，然后以该bean的name作为Map的Key，以bean实例作为对应的Value，从而构建一个Map并注入到依赖处

我们也可以自定义一个端点
```java
@Component
@Endpoint(id = "hello")
public class MyEndpoint {

    @ReadOperation
    public String hello(@Selector String name) {
        return "hello " + name;
    }

}
```
这样我们可以直接通过`http://127.0.0.1:8080/actuator/hello/{name}`来访问这个端点

在尝试JMX访问的时候遇到一个坑，因为我用的Spring Boot 2.2.2版本，在jconsole中并没有找到endpoint，在JmxEndpointDiscoverer中debug也没有进来，然后看到JmxEndpointAutoConfiguration上的注解ConditionalOnProperty发现原来需要打开JMX，默认是false的
```
spring.jmx.enabled=true
```
然后在jconsole中就能看到endpoint节点，并能查看端点信息和操作了
![jmx](jmx.png)
在此之外，配上jolokia，完全兼容并支撑JMX组件，它可以当做JMX-HTTP的桥梁可以让我们通过HTTP访问JMX
```
<dependency>
	<groupId>org.jolokia</groupId>
	<artifactId>jolokia-core</artifactId>
	<version>1.6.2</version>
</dependency>
```
于是就能通过`http://127.0.0.1:8080/actuator/jolokia/read/org.springframework.boot:name=Hello,type=Endpoint`访问了，也可以通过·http://127.0.0.1:8080/actuator/jolokia/list·查看列表






WebEndpointAutoConfiguration创建
WebEndpointDiscoverer    PathMappedEndpoints    	


WebMvcEndpointManagementContextConfiguration创建
WebMvcEndpointHandlerMapping 

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

// metrics

MetricsAutoConfiguration 
MeterRegistryPostProcessor
MeterRegistryConfigurer


# 源码解析

## HealthIndicator

SpringBoot通过HealthIndicatorAutoConfiguration提供了一些常见服务的监控检查支持，比如：DataSourceHealthIndicator、RedisHealthIndicator、SolrHealthIndicator等
![]()
除了SpringBoot提供的检查，还可以通过提供HealthIndicator的实现，注册到IOC容器，让SpringBoot自动发现并使用它们

## endpoints

虽然我们定义了HealthIndicator，但它只是在应用内部，需要将它暴露出去访问才能实现出它的功效，这就需要将endpoints暴露给外部监控者，SpringBoot会议JmxEndpointExporter将所有Endpoint实例以JMX MBean的形式暴露，默认情况下这些Endpoint对应的JMX MBean会放在`org.springframework.boot`命名空间下面，不过可以通过`endpoints.jmx.domain`配置项进行更改，比如 `endpoints.jmx.domain=com.test.management`


JMX和HTTP是endpoints对外开放访问最常用的方式，鉴于Java的序列化漏洞以及JMX的远程访问防火墙问题，因此建议用HTTP并配合安全防护将SpringBoot应用的endpoints开放给外部监控者使用


MBean的名称通常是从端点的id生成的。 例如，health端点暴露为 org.springframework.boot/Endpoint/healthEndpoint。

如果应用程序包含多个Spring ApplicationContext，可能会发现名称发生冲突。 要解决此问题，可以将endpoints.jmx.unique-names属性设置为true，以便MBean名称始终是唯一的


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













# SOFABoot

SOFABoot在SpringBoot的Liveness检查能力的基础上，增加了Readiness检查能力



Kubelet使用liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness探针将捕获到deadlock，重启处于该状态下的容器，使应用程序在存在bug的情况下依然能够继续运行下去（谁的程序还没几个bug呢）。

Kubelet使用readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当Pod中的容器都处于就绪状态时kubelet才会认定该Pod处于就绪状态。该信号的作用是控制哪些Pod应该作为service的后端。如果Pod处于非就绪状态，那么它们将会被从service的load balancer中移除


ReadinessCheckListener


SOFABoot(   search: Readiness Check) 

https://blog.csdn.net/beyondself_77/article/details/80846270
https://blog.csdn.net/weixin_33912638/article/details/87978436















参考：  
[Spring Boot 参考指南（端点）](https://segmentfault.com/a/1190000015309478?utm_source=tag-newest)  

