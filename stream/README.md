# 第七章 消息驱动Spring Cloud Stream

Spring Cloud Stream是一个用来为微服务应用构建消息驱动能力的框架。它通过使用Spring Integration来连接消息代理中间件以实现消息事件驱动。Spring Cloud Stream 为一些供应商的消息中间件产品提供了个性化的自动化配置实现，并且引入了发布-订阅、消费组以及分区这三个核心概念。



Spring Cloud Stream 构建的应用程序与消息中间件之间是通过绑定器Binder相关联的，绑定器对于应用程序而言起到了隔离作用，它使得不同消息中间件的实现细节对应用程序来说是透明的。所以对于每一个Spring Cloud Stream的应用程序来说，它不需要知晓消息中间件的通信细节，它只需知道Binder对应程序提供的抽象概念来使用消息中间件来实现业务逻辑即可，而这个抽象概念就是消息通道：Channel。



Spring Cloud Stream中的消息通信方式遵循了发布-订阅模式，通过共享的Topic主题进行广播。这里所提到的Topic主题是Spring Cloud Stream中的一个抽象概念，在RabbitMQ中它对应Exchange，而在Kafka中则对应Kafka中的Topic。



**消费组** 在现实的微服务框架中，我们的每一个微服务应用为了实现高可用和负载均衡，实际上都会部署多个实例。如果在同一个主题上的应用需要启动多个实例的时候，我们可以通过spring.cloud.stream.bindings.input.group属性为应用指定一个组名，这样这个应用的多个实例在接收到消息的时候，只会有一个成员真正收到消息并进行处理。



**消费分区** 通过引入消费组的概念，我们已经能够在多实例的情况下，保障每个消息只被组内的一个实例消费，但消费组无法控制消息具体被哪个实例消费。而分区概念的引入就是为了解决这样的问题：当生产者将消息数据发送给多个消费者实例时，保证拥有共同特征的消息数据始终是由同一个消费者实例接收和处理。



在Spring Cloud Stream中还支持使用基于**RxJava**的响应式编程来处理消息的输入和输出。



对于那些不直接支持头信息的消息中间件，Spring Cloud Stream 提供了自己的实现机制，它会在消息发出前自动将消息包装进它自定义的消息封装格式中，并加入头信息。而对于那些自身就支持头信息的消息中间件，Spring Cloud Stream 构建的服务可以接收并处理来自非Spring Cloud Stream构建但包含符合规范头信息的应用程序发出的消息。



分别说说RabbitMQ与Kafka的绑定器是如何使用消息中间件中不同概念来实现消息的生产与消费的。

- RabbitMQ绑定器：在RabbitMQ中，通过Exchange交换器来实现Spring Cloud Stream的主题概念，所以消息通道的输入输出目标映射了一个具体的Exchange交换器。而对于每个消费组，则会为对应的Exchange交换器绑定一个Queue队列进行消息收发。
- Kafka绑定器：由于Kafka自身就有Topic概念，所以Spring Cloud Stream的主题直接采用了Kafka的Topic主题概念，每个消费组的通道目标都会直接连接Kafka的主题就行消息收发。

