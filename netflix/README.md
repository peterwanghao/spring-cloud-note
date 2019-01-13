# 第二章 Spring Cloud Netflix

Netflix是一家互联网流媒体播放商,是美国视频巨头。Netflix由于做视频的原因，访问量非常的大，从而促使其技术快速的发展作为背后强力的支撑，也正是如此，Netflix开始把整体的系统往微服务上迁移。



随着Netflix的微服务大规模的应用，公司在技术上毫无保留的把一整套微服务架构核心技术栈开源了出来，叫做Netflix OSS，其依靠开源社区的力量不断的壮大。



Pivotal在Netflix开源的一整套核心技术产品线的同时，做了一系列的封装后来又并入Spring Cloud 大家庭，就变成了Spring Cloud Netflix。



该项目为Spring Boot应用程序提供了Netflix OSS集成，通过对Spring Environment和其他Spring programming model idioms进行自动配置和绑定。通过一些简单的注解，您可以快速启用和配置应用程序内的通用模式，并使用经过生产测试的Netflix组件构建大型分布式系统。提供的模式包括服务发现（Eureka），断路器（Hystrix），智能路由（Zuul）和客户端负载平衡（Ribbon）等。

