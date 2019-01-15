# 第一节 与RabbitMQ集成

Spring Cloud Stream是一个建立在Spring Boot和Spring Integration之上的框架，有助于创建事件驱动或消息驱动的微服务。在本文中，我们将通过一些简单的例子来介绍Spring Cloud Stream的概念和构造。

## 1 Maven依赖
在开始之前，我们需要添加Spring Cloud Stream与RabbitMQ消息中间件的依赖。
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```
同时为支持Junit单元测试，在pom.xml文件中添加
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-support</artifactId>
    <scope>test</scope>
</dependency>
```
## 2 主要概念
微服务架构遵循“智能端点和哑管道”的原则。端点之间的通信由消息中间件（如RabbitMQ或Apache Kafka）驱动。服务通过这些端点或信道发布事件来进行通信。

让我们通过下面这个构建消息驱动服务的基本范例，来看看Spring Cloud Stream框架的一些主要概念。

### 2.1 服务类
通过Spring Cloud Stream建立一个简单的应用，从Input通道监听消息然后返回应答到Output通道。
```
@SpringBootApplication
@EnableBinding(Processor.class)
public class MyLoggerServiceApplication {
	public static void main(String[] args) {
		SpringApplication.run(MyLoggerServiceApplication.class, args);
	}

	@StreamListener(Processor.INPUT)
	@SendTo(Processor.OUTPUT)
	public LogMessage enrichLogMessage(LogMessage log) {
		return new LogMessage(String.format("[1]: %s", log.getMessage()));
	}
}
```

注解@EnableBinding声明了这个应用程序绑定了2个通道：INPUT和OUTPUT。这2个通道是在接口Processor中定义的（Spring Cloud Stream默认设置）。所有通道都是配置在一个具体的消息中间件或绑定器中。

让我们来看下这些概念的定义：

- Bindings — 声明输入和输出通道的接口集合。
- Binder — 消息中间件的实现，如Kafka或RabbitMQ
- Channel — 表示消息中间件和应用程序之间的通信管道
- StreamListeners — bean中的消息处理方法，在中间件的MessageConverter特定事件中进行对象序列化/反序列化之后，将在信道上的消息上自动调用消息处理方法。
- Message Schemas — 用于消息的序列化和反序列化，这些模式可以静态读取或者动态加载，支持对象类型的演变。

将消息发布到指定目的地是由发布订阅消息模式传递。发布者将消息分类为主题，每个主题由名称标识。订阅方对一个或多个主题表示兴趣。中间件过滤消息，将感兴趣的主题传递给订阅服务器。订阅方可以分组，消费者组是由组ID标识的一组订户或消费者，其中从主题或主题的分区中的消息以负载均衡的方式递送。

### 2.2 测试类
测试类是一个绑定器的实现，允许与通道交互和检查消息。让我们向上面的enrichLogMessage 服务发送一条消息，并检查响应中是否包含文本“[ 1 ]：”：
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = MyLoggerServiceApplication.class)
@DirtiesContext
public class MyLoggerApplicationIntegrationTest {
	@Autowired
	private Processor pipe;

	@Autowired
	private MessageCollector messageCollector;

	@Test
	public void whenSendMessage_thenResponseShouldUpdateText() {
		pipe.input().send(MessageBuilder.withPayload(new LogMessage("This is my message")).build());

		Object payload = messageCollector.forChannel(pipe.output()).poll().getPayload();

		assertEquals("[1]: This is my message", payload.toString());
	}
}
```

### 2.3 RabbitMQ配置
我们需要在工程src/main/resources目录下的application.yml文件里增加RabbitMQ绑定器的配置。
```
spring:
  cloud:
    stream:
      bindings:
        input:
          destination: queue.log.messages
          binder: local_rabbit
          group: logMessageConsumers
        output:
          destination: queue.pretty.log.messages
          binder: local_rabbit
      binders:
        local_rabbit:
          type: rabbit
          environment:
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
                virtual-host: /
```

input绑定使用名为queue.log.messages的消息交换机，output绑定使用名为queue.pretty.log.messages的消息交换机。所有的绑定都使用名为local_rabbit的绑定器。
请注意，我们不需要预先创建RabbitmQ交换机或队列。运行应用程序时，两个交换机都会自动创建。

## 3 自定义通道
在上面的例子里，我们使用Spring Cloud提供的Processor接口，这个接口有一个input通道和一个output通道。

如果我们想创建一些不同，比如说一个input通道和两个output通道，可以新建一个自定义处理器。

```
public interface MyProcessor {
	String INPUT = "myInput";

	@Input
	SubscribableChannel myInput();

	@Output("myOutput")
	MessageChannel anOutput();

	@Output
	MessageChannel anotherOutput();
}
```
### 3.1 服务类
Spring将为我们提供这个接口的实现。通道的名称可以通过使用注解来设定，比如@Output(“myOutput”)。如果没有设置的话，Spring将使用方法名来作为通道名称。因此这里有三个通道：myInput， myOutput， anotherOutput。

现在我们可以增加一些路由规则，如果接收到的值小于10则走一个output通道；如果接收到的值大于等于10则走另一个output通道。
```
@Autowired
private MyProcessor processor;

@StreamListener(MyProcessor.INPUT)
public void routeValues(Integer val) {
	if (val < 10) {
		processor.anOutput().send(message(val));
	} else {
		processor.anotherOutput().send(message(val));
	}
}

private static final <T> Message<T> message(T val) {
	return MessageBuilder.withPayload(val).build();
}
```
### 3.2 测试类
发送不同的消息，判断返回值是否是通过不同的通道获得。
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = MultipleOutputsServiceApplication.class)
@DirtiesContext
public class MultipleOutputsServiceApplicationIntegrationTest {
	@Autowired
	private MyProcessor pipe;

	@Autowired
	private MessageCollector messageCollector;

	@Test
	public void whenSendMessage_thenResponseIsInAOutput() {
		whenSendMessage(1);
		thenPayloadInChannelIs(pipe.anOutput(), 1);
	}

	@Test
	public void whenSendMessage_thenResponseIsInAnotherOutput() {
		whenSendMessage(11);
		thenPayloadInChannelIs(pipe.anotherOutput(), 11);
	}

	private void whenSendMessage(Integer val) {
		pipe.myInput().send(MessageBuilder.withPayload(val).build());
	}

	private void thenPayloadInChannelIs(MessageChannel channel, Integer expectedValue) {
		Object payload = messageCollector.forChannel(channel).poll().getPayload();
		assertEquals(expectedValue, payload);
	}
}
```

## 4 根据条件分派
使用@StreamListener 注释，我们还可以使用自定义的SpEL表达式来过滤用户期望的消息。下面这个例子，我们使用条件调度将消息路由到不同的输出。
```
@Autowired
private MyProcessor processor;
 
@StreamListener(
  target = MyProcessor.INPUT, 
  condition = "payload < 10")
public void routeValuesToAnOutput(Integer val) {
    processor.anOutput().send(message(val));
}
 
@StreamListener(
  target = MyProcessor.INPUT, 
  condition = "payload >= 10")
public void routeValuesToAnotherOutput(Integer val) {
    processor.anotherOutput().send(message(val));
}
```

## 5 总结
在本教程中，我们介绍了Spring Cloud Stream的主要概念，并展示了如何通过RabbitMQ上的一些简单示例来使用它。