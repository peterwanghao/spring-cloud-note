# 第三节 客户端调用Ribbon

Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡工具，它基于Netflix Ribbon实现。可轻松地将面向服务的REST请求自动转换成客户端负载均衡的服务调用。Netflix Ribbon是一个在云端使用的IPC（进程间通信）库，主要提供客户端的负载均衡算法。

除了客户端负载均衡算法，Ribbon还提供其他功能：

* __服务发现集成__  - Ribbon负载均衡器可在动态环境（如云）中提供服务发现功能。与Eureka和Netflix服务发现组件的集成包含在Ribbon库中。
* __容错__ - Ribbon API可以动态确定服务是否在工作环境中运行，并且可以检测到服务器是否停机。
* __可配置的负载均衡规则__  - Ribbon支持RoundRobinRule，AvailabilityFilteringRule，WeightedResponseTimeRule等开箱即用的规则，并且还支持自定义规则。



通过Spring Cloud Ribbon的封装，在微服务框架中使用客户端负载均衡调用非常简单，只需要如下两步：

- 服务提供者只需要启动多个服务实例并注册到一个注册中心或是多个相关联的服务注册中心。
- **服务消费者直接通过调用被@LoadBalanced注解修饰过的RestTemplate来实现面向服务的接口调用。**



客户端负载均衡器应具备的几种能力：

- 根据传入的服务名，从**负载均衡器**中挑选一个对应服务的实例。

- 使用从负载均衡器中挑选出的服务实例来执行请求内容。

- 构建一个合适的host：port形式的URI。


下面通过一个简单的例子介绍Ribbon是如何使用的。在本例中我们创建一个简单的微服务应用，使用Spring RestTemplate去调用服务提供者的服务，并配合Ribbon API实现负载均衡。服务提供方启动2个服务实例，使用WeightedResponseTimeRule负载均衡策略，在配置文件中指定服务列表。通过服务名的方式实现在2个服务实例间的负载。

## 1 依赖管理
在pom.xml里增加依赖库
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

## 2 Ribbon配置
Ribbon API支持下面这些内容的配置

* __Rule__ — 逻辑组件，指定我们在应用程序中使用的负载均衡规则。
* __Ping__  — 一个组件，它指定了用于实时确定服务器可用性的机制。
* __ServerList__ — 可以是动态或静态的。在我们的例子中，我们使用的是一个静态的服务器列表，我们直接在应用程序配置文件中定义了它们。

下面是配置类
```
public class RibbonConfiguration {
	@Autowired
	IClientConfig ribbonClientConfig;

	// 判断服务实例是否有效，ping某个url判断其是否alive
	@Bean
	public IPing ribbonPing(IClientConfig config) {
		return new PingUrl();
	}

	// 负载均衡策略，根据服务实例的运行情况来计算权重
	@Bean
	public IRule ribbonRule(IClientConfig config) {
		return new WeightedResponseTimeRule();
	}
}
```

## 3 配置文件
application.yml

```
spring:
  application:
    name: spring-cloud-ribbon

server:
  port: 8888

ping-server:
  ribbon:
    eureka:
      enabled: false
    listOfServers: localhost:8081,localhost:8082
    ServerListRefreshInterval: 15000
```
在上面的文件中定义了

* 应用的名称
* 应用的端口
* 服务列表的名称：ping-server
* 禁用 eureka服务发现组件，不把此客户端发布为服务
* 定义了用于负载平衡的服务器列表，在这里有2个服务器
* 配置服务器的刷新频率

## 4 RibbonClient
现在实现Ribbon客户端。
```
@SpringBootApplication
@RestController
@RibbonClient(name = "ping-a-server", configuration = RibbonConfiguration.class)
public class ServerLocationApp {

	@LoadBalanced
	@Bean
	RestTemplate getRestTemplate() {
		return new RestTemplate();
	}

	@Autowired
	RestTemplate restTemplate;

	@RequestMapping("/server-location")
	public String serverLocation() {
		String servLoc = this.restTemplate.getForObject("http://ping-server/locate", String.class);
		return servLoc;
	}

	public static void main(String[] args) {
		SpringApplication.run(ServerLocationApp.class, args);
	}
}
```
我们定义了一个Controller类通过@RestController注解并添加了@RibbonClient注解。注解里的配置就是前面定义的配置类，在这个类中我们为这个应用程序提供了所需的Ribbon API配置。需注意的是在RestTemplate上使用了@LoadBalanced注解表示使用RestTemplate去调用服务时进行负载均衡。

## 5 服务提供者
一个简单的定位服务。
```
@Configuration
@EnableAutoConfiguration
@RestController
public class TestServiceProvider {

	@RequestMapping(value = "/locate")
	public String locationDetails() {
		return "Beijing";
	}
}
```

## 6 测试

通过一个测试用例启动2个服务实例，然后调用RibbonClient得到定位信息。
```
@SuppressWarnings("unused")
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ServerLocationApp.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ServerLocationAppIntegrationTest {
	ConfigurableApplicationContext application2;
	ConfigurableApplicationContext application3;

	@Before
	public void startApps() {
		this.application2 = startApp(8081);
		this.application3 = startApp(8082);
	}

	@After
	public void closeApps() {
		this.application2.close();
		this.application3.close();
	}

	@LocalServerPort
	private int port;

	@Autowired
	private TestRestTemplate testRestTemplate;

	@Test
	public void loadBalancingServersTest() throws InterruptedException {
		// 调用Ribbon客户端
		ResponseEntity<String> response = this.testRestTemplate
				.getForEntity("http://localhost:" + this.port + "/server-location", String.class);
		assertEquals(response.getBody(), "Beijing");
	}

	// 启动服务实例
	private ConfigurableApplicationContext startApp(int port) {
		return SpringApplication.run(TestServiceProvider.class, "--server.port=" + port, "--spring.jmx.enabled=false");
	}
}
```

##7 总结

在本文中我们介绍了Ribbon API并通过一个简单的应用说明了它是如何实现的。Ribbon API不仅提供了客户端负载均衡算法还内置了一些容错保护，如前所述它可以通过不断的ping服务器来确定服务器的可用性和排除掉不能提供服务的服务器。除此之外，它还实现了断路器模式，根据指定的标准过滤服务器。断路器模式不用等待超时而迅速拒绝请求失败的服务器的请求，从而最大限度地减少服务器故障对性能的影响。