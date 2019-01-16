# 第一节 服务治理Eureka

## 1、概述
微服务框架中最为核心和基础的模块就是服务治理，它主要用来实现各个微服务实例的自动化注册与发现。在这个体系结构中有一个“中心点”——服务注册中心，每个服务必须注册到服务注册中心。而各个服务之间进行通讯并不需要知道具体服务的主机名和端口。这种实现的一个缺点是所有客户机必须实现某种逻辑来与这个中心点进行交互，这样在实现服务请求之前将增加一次额外的网络往返。



Spring Cloud 使用 Netflix Eureka 来实现服务注册与发现，它既包含了服务端组件，也包含了客户端组件。Eureka服务端也称为服务注册中心，支持高可用配置。它依托强一致性提供良好的服务实例可用性。Eureka客户端可以同时充当服务器，将其状态复制到一个连接的对等点上。换句话说，客户机检索服务注册中心所有连接的节点的列表，并通过负载平衡算法向所有其他服务发出请求。每个客户机为了声明自己的存活状态，他们必须向注册中心发送一个心跳信号。

## 2、实现机制

Spring Cloud Eureka实现的服务治理机制强调了CAP原理中的AP，即可用性与可靠性，它与Zookeeper这类强调CP（一致性、可靠性）的服务治理框架最大的区别就是，Eureka为了实现更高的服务可用性，牺牲了一定的一致性，在极端情况下它宁愿接受故障实例也不要丢掉“健康”实例，比如，当服务注册中心的网络发生故障断开时，由于所有的服务实例无法维持续约心跳，在强调CP的服务治理中将会把所有服务实例都剔除掉，而Eureka则会因为超过85%的实例丢失心跳而会触发保护机制，注册中心将会保留此时的所有节点，以实现服务间依然可以进行互相调用的场景，即使其中有部分故障节点，但这样做可以继续保障大多数的服务正常消费。



由于Spring Cloud Eureka在可用性与一致性上的取舍，不论是由于触发了保护机制还是服务剔除的延迟，引起服务调用到故障实例的时候，我们还是希望能够增强对这类问题的容错。所以，我们在实现服务调用的时候通常会加入一些重试机制。



## 3、 实现示例

在本例中为了体现服务治理功能，实现了三个微服务:

- 一个服务注册中心 (Eureka Server)

- 一个在注册中心注册的REST服务(Eureka Client) 

- 一个Web应用程序，服务的消费者(Spring Cloud Netflix Feign Client)


### 3.1 注册中心

使用Spring Cloud Netflix Eureka实现一个注册中心非常简单。在pom里增加spring-cloud-starter-eureka-server依赖，在启动类里添加@EnableEurekaServer注解。

首先创建一个Maven工程，并添加一些依赖关系
```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>1.5.10.RELEASE</version>
	<relativePath />
</parent>       
<dependencies>
	<dependency>
	<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-eureka-server</artifactId>
	</dependency>
</dependencies>
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-parent</artifactId>
			<version>Edgware.SR2</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```
然后创建一个启动类
```
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
在application.yml中加入配置信息
```
server:
  port: 8888 #服务注册中心端口号
  
eureka:
  instance:
    hostname: localhost #服务注册中心实例的主机名
  client:
    registerWithEureka: false #是否向服务注册中心注册自己
    fetchRegistry: false #是否检索服务
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
现在通过浏览器访问http://localhost:8888 可以看到Eureka的控制台，在那里能看到将来注册后的服务实例和一些状态和健康指标。

![Eureka的控制台](https://upload-images.jianshu.io/upload_images/11110195-6914b86ace9cdc30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.2 服务提供者

首先我们添加一些依赖关系
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
然后实现主应用类
```
@SpringBootApplication
@EnableEurekaClient
@RestController
public class EurekaClientApplication implements GreetingController {
	@Autowired
	@Lazy
	private EurekaClient eurekaClient;

	@Value("${spring.application.name}")
	private String appName;

	public static void main(String[] args) {
		SpringApplication.run(EurekaClientApplication.class, args);
	}

	public String greeting() {
		return String.format("Hello from '%s'!", eurekaClient.getApplication(appName).getName());
	}

}
```
在application.yml文件里添加配置
```
spring:
  application:
    name: spring-cloud-eureka-client

server:
  port: 0

eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_URI:http://localhost:8888/eureka}
  instance:
    preferIpAddress: true
```
我们让Spring Boot为我们选择一个随机端口，因为后面是使用名称来访问这个服务。现在重新访问Eureka控制台可以看到新注册的这个服务。

![注册的服务](https://upload-images.jianshu.io/upload_images/11110195-3e2c072bdc506fc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3 服务消费者

最后我们实现第三个微服务，使用Spring Netflix Feign Client实现的REST消费Web应用。

在Feign Client工程的pom.xml文件里添加一些依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```
定义Feign Client 接口，引入要调用的服务名称
```
@FeignClient("spring-cloud-eureka-client")
public interface GreetingClient{
	@RequestMapping("/greeting")
    String greeting();
}
```
定义主服务类，是一个controller控制器
```
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@Controller
public class FeignClientApplication {
	@Autowired
	private GreetingClient greetingClient;

	public static void main(String[] args) {
		SpringApplication.run(FeignClientApplication.class, args);
	}

	@RequestMapping("/get-greeting")
	public String greeting(Model model) {
		model.addAttribute("greeting", greetingClient.greeting());
		return "greeting-view";
	}
}
```
下面是HML模板可进行页面展现
```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
    <head>
        <title>Greeting Page</title>
    </head>
    <body>
        <h2 th:text="${greeting}"></h2>
    </body>
</html>
```
配置文件内容
```
spring:
  application:
    name: spring-cloud-eureka-feign-client

server:
  port: 8080

eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_URI:http://localhost:8888/eureka}
```
启动运行这个服务，在浏览器里访问 http://localhost:8080/get-greeting 就会看到
```
Hello from 'SPRING-CLOUD-EUREKA-CLIENT'!
```
## 4、结论
我们现在可以使用Spring Cloud Netflix Eureka Server实现服务注册中心，并注册一些Eureka Clients。在创建服务提供者的时候没有指定端口，这样服务将监听一个随机选择的端口。使用Feign Client通过服务名可以定位和消费这个REST服务，甚至在位置发生变化时也可以。