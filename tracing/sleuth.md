# 第一节 日志跟踪

Spring Cloud Sleuth是一个在应用中实现日志跟踪的强有力的工具。使用Sleuth库可以应用于计划任务 、多线程服务或复杂的Web请求，尤其是在一个由多个服务组成的系统中。当我们在这些应用中来诊断问题时，即使有日志记录也很难判断出一个请求需要将哪些操作关联在一起。

如果想要诊断复杂操作，通常的解决方案是在请求中传递唯一的ID到每个方法来识别日志。而Sleuth可以与日志框架Logback、SLF4J轻松地集成，通过添加独特的标识符来使用日志跟踪和诊断问题。

## 1. 依赖管理
在Spring Boot Web应用中增加Sleuth非常简单，只需在pom.xml增加以下的依赖：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```
同时添加一个应用名称来识别应用程序的日志，在配置文件application.properties file增加一行
```
spring.application.name=Sleuth Tutorial
```

## 2. 一个简单的WEB应用
首先创建一个服务
```
@Service
public class SleuthService {
    public void doSomeWorkSameSpan() throws InterruptedException {
        Thread.sleep(1000L);
        logger.info("Doing some work");
    }
}
```
然后写一个controller去调用这个服务
```
@RestController
public class SleuthController {
    @Autowired
    private final SleuthService sleuthService;
    @GetMapping("/same-span")
	public String helloSleuthSameSpan() throws InterruptedException {
		logger.info("Same Span");
		sleuthService.doSomeWorkSameSpan();
		return "success";
	}
}
```
启动应用，访问 http://localhost:8080/same-span，查看日志
```
2018-04-16 16:35:54.151  INFO [Sleuth Tutorial,0afe3e67168fce4f,0afe3e67168fce4f,false] 5968 --- [nio-8080-exec-1] c.p.s.cloud.sleuth.SleuthController      : Same Span
2018-04-16 16:35:55.161  INFO [Sleuth Tutorial,0afe3e67168fce4f,0afe3e67168fce4f,false] 5968 --- [nio-8080-exec-1] c.p.spring.cloud.sleuth.SleuthService    : Doing some work
```
日志的格式为：[**application name**, **traceId**, **spanId**, **export**]

- **application name** — 应用的名称，也就是application.properties中的spring.application.name参数配置的属性。
- **traceId** — 为一个请求分配的ID号，用来标识一条请求链路。
- **spanId** — 表示一个基本的工作单元，一个请求可以包含多个步骤，每个步骤都拥有自己的spanId。一个请求包含一个TraceId，多个SpanId
- **export** — 布尔类型。表示是否要将该信息输出到类似Zipkin这样的聚合器进行收集和展示。

可以看到，TraceId和SpanId在两条日志中是相同的，即使消息来源于两个不同的类。这就可以在不同的日志通过寻找traceid来识别一个请求。

## 3. 异步线程池
添加@Async注解实现异步线程调用，增加一个配置类来建立线程池
```
@Configuration
@EnableAsync
public class ThreadConfig extends AsyncConfigurerSupport{
	@Autowired
    private BeanFactory beanFactory;
	
	@Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setCorePoolSize(7);
        threadPoolTaskExecutor.setMaxPoolSize(42);
		threadPoolTaskExecutor.setQueueCapacity(11);
		threadPoolTaskExecutor.setThreadNamePrefix("MyExecutor-");
        threadPoolTaskExecutor.initialize();

        return new LazyTraceExecutor(beanFactory, threadPoolTaskExecutor);
    }
}
```
在这里我们继承了AsyncConfigurerSupport指定了具体的异步执行器，使用LazyTraceExecutor 确保traceId和spanId正确的传递，同时给类加上@EnableAsync 注解。

然后在服务里增加一个异步方法：
```
@Async
public void asyncMethod() throws InterruptedException {
    logger.info("Start Async Method");
    Thread.sleep(1000L);
    logger.info("End Async Method");
}
```
同时在controller里调用此服务
```
@GetMapping("/async")
public String helloSleuthAsync() throws InterruptedException {
	logger.info("Before Async Method Call");
	sleuthService.asyncMethod();
	logger.info("After Async Method Call");
	return "success";
}
```
访问http://localhost:8080/async，查看日志如下：
```
2018-04-16 16:55:06.859  INFO [Sleuth Tutorial,4bc16602ac8262d5,4bc16602ac8262d5,false] 5968 --- [nio-8080-exec-6] c.p.s.cloud.sleuth.SleuthController      : Before Async Method Call
2018-04-16 16:55:06.872  INFO [Sleuth Tutorial,4bc16602ac8262d5,4bc16602ac8262d5,false] 5968 --- [nio-8080-exec-6] c.p.s.cloud.sleuth.SleuthController      : After Async Method Call
2018-04-16 16:55:06.905  INFO [Sleuth Tutorial,4bc16602ac8262d5,4d216310c71e9c22,false] 5968 --- [   MyExecutor-1] c.p.spring.cloud.sleuth.SleuthService    : Start Async Method
2018-04-16 16:55:07.905  INFO [Sleuth Tutorial,4bc16602ac8262d5,4d216310c71e9c22,false] 5968 --- [   MyExecutor-1] c.p.spring.cloud.sleuth.SleuthService    : End Async Method
```
通过例子可以看到Sleuth将traceId传入了异步方法并创建了以新的spanId，代表这是同一个请求但进入了另一个处理阶段，由一个异步线程来执行。

## 4. 计划任务
现在我们来看下Sleuth是怎么在@Scheduled 方法里工作的，修改ThreadConfig 类是它支持时间调度。
```
@Configuration
@EnableAsync
@EnableScheduling
public class ThreadConfig extends AsyncConfigurerSupport
  implements SchedulingConfigurer {
  
    //...
     
    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
        scheduledTaskRegistrar.setScheduler(schedulingExecutor());
    }
 
    @Bean(destroyMethod = "shutdown")
    public Executor schedulingExecutor() {
        return Executors.newScheduledThreadPool(1);
    }
}
```
我们在这里实现了SchedulingConfigurer 接口并重写了configureTasks 方法，同时在类上面添加了@EnableScheduling 注解。

新写一个调度服务，定义了一个每3秒执行一次的计划任务。
```
@Service
public class SchedulingService {
	private Logger logger = LoggerFactory.getLogger(this.getClass());
	private final SleuthService sleuthService;

	@Autowired
	public SchedulingService(SleuthService sleuthService) {
		this.sleuthService = sleuthService;
	}

	@Scheduled(fixedDelay = 30000)
	public void scheduledWork() throws InterruptedException {
		logger.info("Start some work from the scheduled task");
		sleuthService.asyncMethod();
		logger.info("End work from scheduled task");
	}
}
```
重启应用等待任务的执行，观察日志：
```
2018-04-16 17:10:20.232  INFO [Sleuth Tutorial,5d891f56b6f18505,5d891f56b6f18505,false] 7936 --- [pool-1-thread-1] c.p.s.cloud.sleuth.SchedulingService     : Start some work from the scheduled task
2018-04-16 17:10:20.232  INFO [Sleuth Tutorial,5d891f56b6f18505,5d891f56b6f18505,false] 7936 --- [pool-1-thread-1] c.p.s.cloud.sleuth.SchedulingService     : End work from scheduled task
2018-04-16 17:10:20.232  INFO [Sleuth Tutorial,5d891f56b6f18505,abe8b45c36f93be8,false] 7936 --- [   MyExecutor-2] c.p.spring.cloud.sleuth.SleuthService    : Start Async Method
2018-04-16 17:10:21.232  INFO [Sleuth Tutorial,5d891f56b6f18505,abe8b45c36f93be8,false] 7936 --- [   MyExecutor-2] c.p.spring.cloud.sleuth.SleuthService    : End Async Method
2018-04-16 17:10:50.235  INFO [Sleuth Tutorial,0dd33d0764f0e6db,0dd33d0764f0e6db,false] 7936 --- [pool-1-thread-1] c.p.s.cloud.sleuth.SchedulingService     : Start some work from the scheduled task
2018-04-16 17:10:50.235  INFO [Sleuth Tutorial,0dd33d0764f0e6db,0dd33d0764f0e6db,false] 7936 --- [pool-1-thread-1] c.p.s.cloud.sleuth.SchedulingService     : End work from scheduled task
2018-04-16 17:10:50.236  INFO [Sleuth Tutorial,0dd33d0764f0e6db,5062d156f65a293e,false] 7936 --- [   MyExecutor-3] c.p.spring.cloud.sleuth.SleuthService    : Start Async Method
2018-04-16 17:10:51.236  INFO [Sleuth Tutorial,0dd33d0764f0e6db,5062d156f65a293e,false] 7936 --- [   MyExecutor-3] c.p.spring.cloud.sleuth.SleuthService    : End Async Method
```
在这里可以看到Sleuth为每个任务实例都创建一个新的traceId和spanId。

## 5. 总结
通过本文我们看到Spring Cloud Sleuth可以应用在各种各样的单一Web应用中。我们可以使用这项技术轻松地为一个请求采集日志，即使请求跨越多个线程。帮助我们在多线程环境下进行清晰的调试，通过识别traceId和spanId来确定每一个操作和操作中的每一步，这样可以减轻我们做日志分析的复杂性。

除了在单一应用外，Spring Cloud Sleuth为微服务架构中的分布式服务跟踪问题提供了一套完整的解决方案。可应用于通过RestTemplate发起的请求，通过Zuul代理传递的请求，通过诸如RabbitMQ、Kafka（或者其他任何Spring Cloud Stream绑定器实现的消息中间件）转递的请求。