# Zuul和Eureka的负载均衡示例

## 1.概述
在本文中，我们将介绍如何通过Zuul和Eureka一起使用来实现负载均衡。

我们将请求路由到注册在Spring Cloud Eureka，并通过Zuul Proxy来发现的REST服务。

## 2.初始设置
我们需要设置Eureka服务器/客户端，如文章[介绍微服务中服务治理Spring-Cloud-Netflix-Eureka](https://blog.csdn.net/peterwanghao/article/details/79587782)所示。

## 3.配置Zuul
Zuul还从Eureka服务站点获取服务列表并进行服务器端负载平衡。
### 3.1Maven配置
首先，我们将为我们的pom.xml添加Zuul Server和Eureka依赖项：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
 </dependency>
```

### 3.2与Eureka整合
其次，我们将在Zuul的application.properties文件中添加必要的属性：
```
server.port=8762
spring.application.name=zuul-server
eureka.instance.preferIpAddress=true
eureka.client.registerWithEureka=true
eureka.client.fetchRegistry=true
eureka.serviceurl.defaultzone=http://localhost:8761/eureka/
```

在这里，我们告诉Zuul在Eureka注册自己的服务，并在8762端口运行。

接下来，我们将使用@EnableZuulProxy和@EnableDiscoveryClient实现主类。使用@EnableZuulProxy将此指示为Zuul Server，并且用@EnableDiscoveryClient将此指示为Eureka客户端：
```
@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class ZuulApplication {
	public static void main(String[] args) {
		SpringApplication.run(ZuulApplication.class, args);
	}
}
```

我们将浏览器指向[http://localhost:8762/routes](http://localhost:8762/routes)。这应该显示Eureka发现的Zuul可用的所有路线：
```
{"/spring-cloud-eureka-client/**":"spring-cloud-eureka-client"}
```

现在，我们将使用获得的Zuul Proxy路由与Eureka客户端进行通信。将我们的浏览器指向[http://localhost:8762/spring-cloud-eureka-client/greeting](http://localhost:8762/spring-cloud-eureka-client/greeting)应该生成如下响应：
```
Hello from 'SPRING-CLOUD-EUREKA-CLIENT with Port Number 8081'!
```

## 4.使用Zuul加载平衡
当Zuul收到请求时，它会获取一个可用的物理位置，并将请求转发给实际的服务实例。Zuul缓存服务实例的位置并将请求转发到实际位置的整个过程是开箱即用的，不需要额外的配置。

在这里，我们可以看到Zuul如何封装同一服务的三个不同实例：
![](https://upload-images.jianshu.io/upload_images/11110195-fe5639ad14dd3104.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在内部，Zuul使用Netflix功能区从服务发现（Eureka Server）中查找服务的所有实例。让我们在出现多个实例时观察这种行为。

### 4.1注册多个实例
我们将从运行两个实例（8081和8082端口）开始。

一旦所有实例都启动，我们可以在日志中观察实例的物理位置在DynamicServerListLoadBalancer中注册，并且路由映射到Zuul Controller，后者负责将请求转发到实际实例：
```
17:23:36.983 [http-nio-8762-exec-4] INFO  c.n.l.DynamicServerListLoadBalancer - DynamicServerListLoadBalancer for client 
spring-cloud-eureka-client initialized: DynamicServerListLoadBalancer:{NFLoadBalancer:name=spring-cloud-eureka-client,
current list of Servers=[192.168.8.234:8081, 192.168.8.234:8082],Load balancer stats=Zone stats: {defaultzone=[Zone:defaultzone;	
Instance count:2;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
```

通过Eureka的控制台，可以看到注册后的服务实例。
![](https://upload-images.jianshu.io/upload_images/11110195-250057df2be5f554.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 4.2负载平衡示例
让我们将浏览器导航到[http://localhost:8762/spring-cloud-eureka-client/greeting](http://localhost:8762/spring-cloud-eureka-client/greeting)。
刷新几次。每一次，我们都应该看到略有不同的结果：
```
Hello from 'SPRING-CLOUD-EUREKA-CLIENT with Port Number 8081'!

Hello from 'SPRING-CLOUD-EUREKA-CLIENT with Port Number 8082'!

Hello from 'SPRING-CLOUD-EUREKA-CLIENT with Port Number 8081'!
```

Zuul收到的每个请求都以循环方式转发给不同的实例。

## 5.结论
正如我们所见，Zuul为Rest服务的所有实例提供了一个URL，并进行负载均衡以将请求以循环方式转发到其中一个实例。
