# Spring Cloud Hystrix断路器例子

## 1.概述
在本文中我们将介绍Spring Cloud Netflix Hystrix - 断路器。我们将使用该库来实现断路器的企业模式，该模式描述了应用程序中不同级别的故障级联策略。

断路器的工作原理类似于电子产品：Hystrix 用来观察正在调用某些相关服务的方法。如果出现了故障，它将打开断路器并将呼叫转发到回退方法。

该库将设置失败次数的阈值，正常情况下它将使电路保持开放状态。当超过容忍失败次数的阈值后，它会将所有后续调用转发给回退方法，以防止将来出现故障。这为相关服务创建了一个时间缓冲区，以便从其失败状态中恢复。

## 2. 服务提供者
要创建演示断路器模式的场景，我们首先需要一个服务。我们将其命名为“ REST Producer”，因为它为启用Hystrix的“ REST消费者 ” 提供数据。

让我们使用spring-boot-starter-web依赖创建一个新的Maven项目：
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

项目本身有意保持简单。它包括了一个控制器接口，里面的GET方法使用@RequestMapping注解绑定了访问地址，此方法简单地返回一个字符串；一个@RestController实现此接口和一个@SpringBootApplication。

首先是服务接口
```
public interface GreetingController {
	@RequestMapping("/greeting/{username}")
	String greeting(@PathVariable("username") String username);
}
```

然后是实现类
```
@RestController
public class GreetingControllerImpl implements GreetingController {
	@Override
	public String greeting(@PathVariable("username") String username) {
		return String.format("Hello %s!\n", username);
	}

}
```

接下来是主要的应用程序类
```
@SpringBootApplication
public class RestProducerApplication {
	public static void main(String[] args) {
		SpringApplication.run(RestProducerApplication.class, args);
    }
}
```

此外，我们定义一个应用程序名称，以便能够从我们稍后将介绍的客户端应用程序中查找我们的“ REST Producer ”。让我们创建一个包含以下内容的application.properties：
```
spring.application.name=rest-producer
server.port=9090
```

现在我们可以使用curl 测试我们的' REST Producer '了：
```
$> curl http://localhost:9090/greeting/Peter
Hello Peter!
```

## 3.使用Hystrix的服务消费者
对于我们的演示场景，我们将实现一个Web应用程序，它使用RestTemplate和Hystrix调用上一步中的REST服务。为简单起见，我们将其称为“ REST消费者 ”。
因此，我们创建了一个新的Maven项目，其中包含spring-cloud -starter- hystrix ，spring-boot-starter-web和spring-boot-starter-thymeleaf作为依赖项：

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

要使断路器工作，Hystix将扫描@Component或@Service注解类以获取@HystixCommand注解方法，为其实现代理并监视其调用。

我们将首先创建一个@Service类，它将被注入@Controller。由于我们正在使用Thymeleaf构建Web应用程序，因此我们还需要一个HTML模板作为视图。
同时在注入@Service的类中实现@HystrixCommand以及相关的回退方法。
```
@Service
public class GreetingService {
	@HystrixCommand(fallbackMethod = "defaultGreeting")
	public String getGreeting(String username) {
		return new RestTemplate().getForObject("http://localhost:9090/greeting/{username}", String.class, username);
	}

	@SuppressWarnings("unused")
	private String defaultGreeting(String username) {
		return "Hello User!";
	}
}
```

RestConsumerApplication将是我们的主要应用程序类。该@EnableCircuitBreaker注解将扫描任何兼容的类路径断路器实现。
```
@SpringBootApplication
@EnableCircuitBreaker
public class RestConsumerApplication {
	public static void main(String[] args) {
		SpringApplication.run(RestConsumerApplication.class, args);
    }
}
```

我们将使用GreetingService设置控制器：
```
@Controller
public class GreetingController {
	@Autowired
	private GreetingService greetingService;

	@RequestMapping("/get-greeting/{username}")
	public String getGreeting(Model model, @PathVariable("username") String username) {
		model.addAttribute("greeting", greetingService.getGreeting(username));
		return "greeting-view";
	}
}
```

这是HTML模板：
```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
    <head>
        <title>Greetings from Hystrix</title>
    </head>
    <body>
        <h2 th:text="${greeting}"/>
    </body>
</html>
```

为确保应用程序正在侦听已定义的端口，我们将以下内容放在application.properties文件中：
```
server.port=8080
```

为了查看正在运行的Hystix断路器，我们将启动“ REST Consumer ”并用浏览器访问http://localhost:8080/get-greeting/Peter 。在正常情况下，将显示以下内容：
```
Hello Peter!
```

为了模拟我们的' REST Producer ' 的失败，我们停止它然后刷新浏览器，我们应该会看到一个通用消息，从@Service中的fallback方法返回：
```
Hello User!
```

## 4.使用Hystrix和Feign的REST消费者
现在，我们将使用Spring Netflix Feign作为声明性REST客户端，而不是Spring RestTemplate。

这种方法的优点是我们以后能够轻松地重构我们的Feign Client界面，以使用Spring Netflix Eureka进行服务发现。

为了启动新项目，我们将制作“ REST Consumer ” 的副本，并添加spring-cloud-starter-feign作为依赖项：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

现在，我们可以使用GreetingController来扩展Feign客户端。@FeignClient的name属性是必需的。如果给出此属性，则可以使用Eureka客户端或URL通过服务发现来查找应用程序。
```
@FeignClient(name = "rest-producer", url = "http://localhost:9090", fallback = GreetingClientFallback.class)
public interface GreetingClient {
	@RequestMapping("/greeting/{username}")
	String greeting(@PathVariable("username") String username);
}
```

我们将Hystrix回退实现为使用@Component注解的类。
```
@Component
public class GreetingClientFallback implements GreetingClient {

	@Override
	public String greeting(@PathVariable("username") String username) {
		return "Hello User!";
	}

}
```

在RestConsumerFeignApplication中，我们将添加一个额外的注解，以实现Feign集成，添加@EnableFeignClients到主应用程序类：
```
@SpringBootApplication
@EnableFeignClients
@EnableCircuitBreaker
public class RestConsumerFeignApplication {
	public static void main(String[] args) {
		SpringApplication.run(RestConsumerFeignApplication.class, args);
    }
}
```

我们将修改控制器以使用自动连接的Feign Client而不是之前注入的@Service来访问greeting服务：
```
@Controller
public class GreetingController {
	@Autowired
	private GreetingClient greetingClient;

	@RequestMapping("/get-greeting/{username}")
	public String getGreeting(Model model, @PathVariable("username") String username) {
		model.addAttribute("greeting", greetingClient.greeting(username));
		return "greeting-view";
	}
}
```

最后，我们将测试这个Feign-enabled的 ' REST Consumer '，就像上一节中的那样。预期结果应该是相同的。

## 5. Hystrix仪表板
Hystrix的一个不错的可选功能是能够在仪表板上监控其状态。

为了实现它，我们将spring-cloud-starter-hystrix-dashboard和spring-boot-starter-actuator放在我们的' REST Consumer ' 的pom.xml中：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
前者需要通过使用@EnableHystrixDashboard注解来启用，后者会自动在我们的Web应用程序中启用所需的指标。

在我们重新启动应用程序之后，我们在浏览器中访问http://localhost:8080/hystrix ，输入'http://localhost:8080/hystrix.stream ' 的指标URL 并开始监控。

最后，我们应该看到这样的事情：
![这里写图片描述](https://img-blog.csdn.net/20180816142900166?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BldGVyd2FuZ2hhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

监视' hystrix.stream '是好的，但是如果你必须观看多个启用Hystrix的应用程序，它将变得不方便。为此，Spring Cloud提供了一个名为Turbine的工具，它可以聚合流以在一个Hystrix仪表板中显示。

## 6.结论
正如我们目前所见，我们现在能够使用Spring Cloud Netflix Hystrix以及Spring RestTemplate或Spring Cloud Netflix Feign实现断路器模式。

这意味着我们能够使用“静态”或“默认”数据来使用包含回退的服务，并且我们能够监控此数据的使用情况。