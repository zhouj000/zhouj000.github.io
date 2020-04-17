---
layout:     post
title:      "Spring Cloud(一) "
date:       2020-01-05
author:     "ZhouJ000"
header-img: "img/in-post/2020/post-bg-2020-headbg.jpg"
catalog: true
tags:
    - Spring Cloud
--- 




# Eureka

spring.factires:
EurekaServerAutoConfiguration
	import: EurekaServerInitializerConfiguration


服务续约

InstanceInfoReplicator#run
DiscoveryClient#register
RetryableEurekaHttpClient#execute
	new RedirectingEurekaHttpClient #execute


	
注册
InstanceRegistry#register	
PeerAwareInstanceRegistryImpl#register
AbstractInstanceRegistry#register














