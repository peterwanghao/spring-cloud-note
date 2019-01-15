# 第五章 服务跟踪

使用Spring Cloud Sleuth为微服务架构增加分布式服务跟踪的能力。



分布式系统中的服务跟踪在理论上并不复杂，它主要包括下面两个关键点：

- 为了实现请求跟踪，当请求发送到分布式系统的入口端点时，只需要服务跟踪框架为该请求创建一个唯一的跟踪标识，同时在分布式系统内部流转的时候，框架始终保持传递该唯一标识，直到返回给请求方为止，这个唯一标识就是前文中提到的Trace ID。通过Trace ID的纪录，我们就能将所有请求过程中的日志关联起来。
- 为了统计各处理单元的时间延迟，当请求到达各个服务组件时，或是处理逻辑到达某个状态时，也通过一个唯一标识来标记它的开始、具体过程以及结束，该标识就是前文中提到的Span ID。对于每个Span来说，它必须有开始和结束两个节点，通过记录开始Span和结束Span的时间戳，就能统计出该Span的时间延迟，除了时间戳记录之外，它还可以包含一些其他元数据，比如事件名称、请求信息等。



在Spring Boot应用中，通过在工程中引入spring-cloud-starter-sleuth依赖之后，它会自动为当前应用构建起各通信通道的跟踪机制，比如：

- 通过诸如RabbitMQ、Kafka（或者其他任何Spring Cloud Stream绑定器实现的消息中间件）转递的请求。
- 通过Zuul代理传递的请求。
- 通过RestTemplate发起的请求。



日志的格式为：**[application name, traceId, spanId, export]**

- **application name —** 应用的名称，也就是application.properties中的spring.application.name参数配置的属性。
- **traceId —** 为一个请求分配的ID号，用来标识一条请求链路。
- **spanId —** 表示一个基本的工作单元，一个请求可以包含多个步骤，每个步骤都拥有自己的spanId。一个请求包含一个TraceId，多个SpanId。
- **export** **—** 布尔类型。表示是否要将该信息输出到类似Zipkin这样的聚合器进行收集和展示。