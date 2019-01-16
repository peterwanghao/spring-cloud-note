# 第二节 集成Zookeeper

本节将介绍如何使用Zookeeper在微服务框架中实现服务发现，该服务发现机制可作为云服务的注册中心。通过Spring Cloud Zookeeper为应用程序提供一种Spring Boot集成，将Zookeeper通过自动配置和绑定 的方式集成到Spring环境中。

在本例子中我们将创建两个应用程序：

*   提供服务的应用程序（称为 服务提供者）
*   使用此服务的应用程序（称为 服务消费者）

Apache Zookeeper将充当我们服务发现设置中的协调者。Apache Zookeeper安装说明可在以下链接中找到。《[zookeeper安装和使用 windows环境](https://blog.csdn.net/tlk20071/article/details/52028945)》

## 1 服务提供者

我们将创建一个服务提供者，通过增加pring-cloud-starter-zookeeper-discovery依赖和在主程序中引入@EnableDiscoveryClient注释。服务的具体内容是通过GET请求返回“Hello World！”字符串。

### 1.1 Maven依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
	<exclusions>
		<exclusion>
			<artifactId>commons-logging</artifactId>
			<groupId>commons-logging</groupId>
		</exclusion>
	</exclusions>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
### 1.2 SpringBoot主程序
使用@EnableDiscoveryClient注释我们的主类，这将使HelloWorld  应用程序自动发布。
```
@SpringBootApplication
@EnableDiscoveryClient
public class HelloWorldApplication {
	public static void main(String[] args) {
		SpringApplication.run(HelloWorldApplication.class, args);
	}
}
```
一个简单的Controller
```
@RestController
public class HelloWorldController {
	@GetMapping("/helloworld")
	public String HelloWorld() {
		return "Hello World!";
	}
}
```
### 1.3 配置文件
在application.yml中定义服务名称用来被客户端调用，同时配置Zookeeper信息用来注册服务。
```
spring:
  application:
    name: HelloWorld
  cloud:
    zookeeper:
      connect-string: localhost:2181
      discovery:
        enabled: true
server:
  port: 8081
endpoints:
  restart:
    enabled: true
logging:
  level:
    org.apache.zookeeper.ClientCnxn: WARN
```
## 2 服务消费者

现在我们来创建一个REST服务消费者，它使用spring Netflix Feign Client来调用服务。

### 2.1 Maven依赖
在pom文件里增加需要依赖的一些组件包。
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
	<exclusions>
		<exclusion>
			<artifactId>commons-logging</artifactId>
			<groupId>commons-logging</groupId>
		</exclusion>
	</exclusions>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```
### 2.2 SpringBoot主程序
同服务提供者一样在主程序中增加@EnableDiscoveryClient 注解。
```
@SpringBootApplication
@EnableDiscoveryClient
public class GreetingApplication {
	public static void main(String[] args) {
		SpringApplication.run(GreetingApplication.class, args);
	}
}
```
### 2.3 Controller类
以下是一个简单的服务控制器类，它通过注入接口helloWorldClient对象调用服务提供者的接口来消费该服务，并在响应中显示它的返回值。
```
@RestController
public class GreetingController {
	@Autowired
	private HelloWorldClient helloWorldClient;

	@GetMapping("/get-greeting")
	public String greeting() {

		return helloWorldClient.HelloWorld();

	}
}
```
### 2.4 声明式服务调用客户端
通过引入spring-cloud-starter-feign组件包使用声明式服务调用方式调用远程服务，使用@FeignClient（“service-name”）注释一个接口并将它自动连接到我们的应用程序中，以便我们以编程方式访问此服务。
```
@Configuration
@EnableFeignClients
@EnableDiscoveryClient
public class HelloWorldClient {
	@Autowired
	private TheClient theClient;

	@FeignClient(name = "HelloWorld")
	interface TheClient {

		@RequestMapping(path = "/helloworld", method = RequestMethod.GET)
		@ResponseBody
		String HelloWorld();
	}

	public String HelloWorld() {
		return theClient.HelloWorld();
	}
}
```
### 2.5 配置文件
```
spring:
  application:
    name: Greeting
  cloud:
    zookeeper:
      connect-string: localhost:2181
server:
  port: 8083
logging:
  level:
    org.apache.zookeeper.ClientCnxn: WARN
```
## 3 运行

HelloWorld REST服务在Zookeeper中注册了自己，Greeting服务通过声明式客户端发现和调用HelloWorld 服务。

现在我们可以运行这两个服务，然后在浏览器中访问 [http://localhost:8083/get-greeting](http://localhost:8083/get-greeting)，将返回
```
Hello World!
```

## 4 总结

在本文中我们看到了如何使用Spring Cloud Zookeeper实现服务发现，并且在Zookeeper中注册了一个名为Hello World的服务。然后通过声明式服务调用方式实现了一个服务消费者Greeting来发现和使用该服务。

在这里介绍下Zookeeper与Eureka这两种服务治理框架的区别。Spring Cloud Eureka实现的服务治理机制强调了CAP原理中的AP，即可用性与可靠性，而Zookeeper这类强调CP（一致性、可靠性）。Eureka为了实现更高的服务可用性，牺牲了一定的一致性，在极端情况下它宁愿接受故障实例也不要丢掉“健康”实例，比如，当服务注册中心的网络发生故障断开时，由于所有的服务实例无法维持续约心跳，在强调CP的服务治理中将会把所有服务实例都剔除掉，而Eureka则会触发保护机制，保留此时的所有节点，以实现服务间依然可以进行互相调用的场景。