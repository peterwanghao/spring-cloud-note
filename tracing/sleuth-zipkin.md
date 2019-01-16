# 第二节 整合Zipkin进行服务跟踪

在这篇文章里介绍如何将Zipkin整合到我们的云服务里。Zipkin是一个开源项目，它提供了在分布式环境下发送、接收、存储和可视化跟踪的机制。这使我们能够对服务之间的活动进行关联，并更清楚地了解我们服务中发生的情况。

这篇文章中的示例工程是基于之前文章的进一步开发，想了解更多情况可阅读["使用Spring Cloud搭建的微服务示例"](https://blog.csdn.net/peterwanghao/article/details/80087088) 。

## 1. Zipkin服务
Zipkin服务将存储我们服务的所有操作步骤，每一步操作都会发送到该服务器用来进行跟踪识别。

### 1.1 创建
新建一个Spring Boot工程，在POM中添加一些依赖
```
<dependency>
	<groupId>io.zipkin.java</groupId>
	<artifactId>zipkin-server</artifactId>
</dependency>
<dependency>
	<groupId>io.zipkin.java</groupId>
	<artifactId>zipkin-autoconfigure-ui</artifactId>
	<scope>runtime</scope>
</dependency>
```
### 1.2 启用
在主应用类里增加@EnableZipkinServer注解，此注解标识这个服务监听所有操作和行为以供查询。
```
@SpringBootApplication
@EnableEurekaClient
@EnableZipkinServer
public class ZipkinApplication {
	public static void main(String[] args) {
		SpringApplication.run(ZipkinApplication.class, args);
	}
}
```

### 1.3 配置
在工程的src/main/resources目录下新建一个bootstrap.properties文件，添加一些配置项。
```
spring.cloud.config.name=zipkin
spring.cloud.config.discovery.service-id=config
spring.cloud.config.discovery.enabled=true
spring.cloud.config.username=configUser
spring.cloud.config.password=configPassword

eureka.client.serviceUrl.defaultZone=http://discUser:discPassword@localhost:8082/eureka/
```

在Git仓库里新增zipkin.properties文件，定义服务名称和端口。
```
spring.application.name=zipkin
server.port=8088

eureka.client.region = default
eureka.client.registryFetchIntervalSeconds = 5

logging.level.org.springframework.web=debug
```

### 1.4 运行
至此Zipkin服务已经创建完毕，我们可以将服务启动起来验证一下。首先需要启动config、discovery服务，然后再启动zipkin服务。在浏览器里输入[http://localhost:8088](http://localhost:8088)，能看到Zipkin的首页。

![Zipkin控制台](./static/zipkin.png)

## 2. 服务配置
将我们的服务配置为Zipkin Client将操作信息发送到Zipkin Server中。这里我们要修改book和gatew服务。

### 2.1 POM引用
在pom.xml文件里添加spring-cloud-starter-zipkin。
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

### 2.2 book服务配置
修改配置库里的book-service.properties文件
```
spring.zipkin.locator.discovery.enabled=true
spring.zipkin.baseUrl=http://localhost:8088/
spring.sleuth.sampler.percentage=1.0
spring.sleuth.web.skipPattern=(^cleanup.*)
```

### 2.3 gateway服务配置
```
spring.zipkin.locator.discovery.enabled=true
spring.zipkin.baseUrl=http://localhost:8088/
spring.sleuth.sampler.percentage=1.0
spring.sleuth.web.skipPattern=(^cleanup.*|.+favicon.*)
```

## 3. 运行
将所有服务的启动起来，通过浏览器访问[http://localhost:8080/book-service/books](http://localhost:8080/book-service/books)

这时在Zipkin控制台[http://localhost:8088](http://localhost:8088) 就能看到服务运行的样子。一个请求所花费的时间以及这次请求包含了2步各花费的时间。

![Zipkin控制台](./static/zipkin-2.png)

点击进去还能看到详细数据。

![Zipkin控制台详情](./static/zipkin-3.png)

![Zipkin控制台详情2](./static/zipkin-4.png)

## 4. 总结
我们已经看到了将Zipkin集成到云应用程序中是多么容易。随着我们的应用越来越复杂，Zipkin可以为我们提供关于一个请求所花费时间的详细信息。这可以帮助我们确定程序在哪里减速，并指出我们的应用需要改进的地方。