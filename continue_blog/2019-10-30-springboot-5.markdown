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
相关actuator其实还引入了`spring-boot-actuator-autoconfigure`和`spring-boot-actuator`。而`spring-boot-starter-actuator`只是个空项目，作为一个统一的引用

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
在springBoot2.0以前，有许多监控端点是打开的，比如/env等。但是2.0以后：  
1、要通过actuator暴露端点，必须同时是启用(enabled)和暴露(exposed)的  
2、所有除了/health和/info的端点，默认都是不暴露的  
3、所有除了/shutdown的端点，默认都是启用的(enabled)  
4、生产环境由于安全性的问题，注意不要暴露敏感端点

通过`application.properties`或`application.ymal`配置可以打开这些监控端点：
```
// 启用指定监控端点，只有shutdown是默认未启用的
management.endpoint.shutdown.enabled=true

// 只暴露/排除指定监控端点，可以使用*表示全部
// 全部暴露，除了env
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env

// 修改访问根路径
management.endpoints.web.base-path=/monitor
```


# 源码解析






HealthIndicatorAutoConfiguration


## AbstractHealthIndicator

## HealthIndicator

endpoints

JmxEndpointExporter




接下来看`spring-boot-actuator-autoconfigure`，查看它的spring.factories，可以看到配置了许多配置类


http://c.biancheng.net/view/4666.html














SOFABoot(   search: Readiness Check) 

https://blog.csdn.net/beyondself_77/article/details/80846270
https://blog.csdn.net/weixin_33912638/article/details/87978436


