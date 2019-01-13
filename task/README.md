# 第八章 Spring Cloud Task

# 1. 概述
Spring Cloud Task的目标是为Spring Boot应用程序提供创建短运行期微服务的功能。在Spring Cloud Task中，我们可以灵活地动态运行任何任务，按需分配资源并在任务完成后检索结果。Tasks是Spring Cloud Data Flow中的一个基础项目，允许用户将几乎任何Spring Boot应用程序作为一个短期任务执行。

# 2. 一个简单的任务应用程序

## 2.1 添加相关依赖项
首先，我们可以添加具有spring-cloud-task-dependencies的依赖关系管理部分：
~~~
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-task-dependencies</artifactId>
    <version>1.2.2.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
~~~
此依赖关系管理通过import范围管理具有依赖关系的版本。

我们需要添加以下依赖项：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-task</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-task-core</artifactId>
</dependency>
```
现在，要启动我们的Spring Boot应用程序，我们需要与相关父级一起使用spring-boot-starter。

我们将使用Spring Data JPA作为ORM工具，因此我们还需要为其添加依赖项：
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```
## 2.2 @EnableTask注解
要引导Spring Cloud Task的功能，我们需要添加@EnableTask注解：
```
@SpringBootApplication
@EnableTask
public class TaskDemo {
    // ...
}
```

该注解在程序中引入了SimpleTaskConfiguration类，该类依次注册TaskRepository及其基础结构。默认情况下，内存映射用于存储TaskRepository的状态。

TaskRepository的主要信息在TaskExecution类中建模。该类的包含的字段有taskName，startTime，endTime，exitMessage。exitMessage在退出的时候存储一些有用信息。

如果退出是由应用程序的任何事件中的失败引起的，则完整的异常堆栈跟踪将存储在此处。

Spring Boot提供了一个接口ExitCodeExceptionMapper，它将未捕获的异常映射到允许经过详细调试的退出代码。Cloud Task将信息存储在数据源中以供将来分析。

## 2.3 为TaskRepository配置DataSource
一旦任务结束，存储TaskRepository的内存映射将消失，我们将丢失与Task事件相关的数据。要想永久存储，我们将使用MySQL作为Spring Data JPA的数据源。

数据源 在application.yml文件中配置。要配置Spring Cloud Task以使用提供的数据源作为TaskRepository的存储，我们需要创建一个扩展DefaultTaskConfigurer的类。

现在，我们可以将配置的Datasource作为构造函数参数发送到超类的构造函数：
```
public class HelloWorldTaskConfigurer extends DefaultTaskConfigurer{
    public HelloWorldTaskConfigurer(DataSource dataSource){
        super(dataSource);
    }
}
```
为了实现上述配置，我们需要使用@Autowired批注注释DataSource的实例，并将实例注入上面定义的HelloWorldTaskConfigurer bean的构造参数中 ：
```
@Autowired
private DataSource dataSource;
@Bean
public HelloWorldTaskConfigurer getTaskConfigurer() {
    return new HelloWorldTaskConfigurer(dataSource);
}
```
这样就完成了将TaskRepository存储到MySQL数据库的配置。

## 2.4 实现
在Spring Boot中，我们可以在应用程序完成启动之前执行任何任务。我们可以使用ApplicationRunner或CommandLineRunner接口来创建一个简单的Task。

我们需要实现这些接口的run方法，并将实现类声明为bean：
```
@Component
	public static class HelloWorldApplicationRunner implements ApplicationRunner {

		@Override
		public void run(ApplicationArguments arg0) throws Exception {
			// TODO Auto-generated method stub
			LOGGER.info("Hello World from Spring Cloud Task!");
		}
	}
```

# 3.  Spring Cloud Task的生命周期
首先，我们在TaskRepository中创建一个条目。这表明所有bean都已准备好在Application中使用，并且Runner接口的run方法已准备好执行。

完成run方法的执行或ApplicationContext事件的任何失败后，TaskRepository将使用另一个条目进行更新。

在任务生命周期中，我们可以在TaskExecutionListener接口中注册侦听器。我们需要一个实现接口的类，它有三个方法 - 在Task的各个事件中触发onTaskEnd，onTaksFailed和onTaskStartup。
```
public class TaskListener implements TaskExecutionListener {

	private final static Logger LOGGER = Logger.getLogger(TaskListener.class.getName());

	@Override
	public void onTaskEnd(TaskExecution arg0) {
		// TODO Auto-generated method stub
		LOGGER.info("End of Task");
	}

	@Override
	public void onTaskFailed(TaskExecution arg0, Throwable arg1) {
		// TODO Auto-generated method stub

	}

	@Override
	public void onTaskStartup(TaskExecution arg0) {
		// TODO Auto-generated method stub
		LOGGER.info("Task Startup");
	}
}
```
我们需要在TaskDemo类中声明实现类的bean ：
```
@Bean
public TaskListener taskListener() {
    return new TaskListener();
}
```
运行TaskDemo类，在控制台可看到任务被执行：
```
14:23:29.974 [main] INFO  o.s.j.e.a.AnnotationMBeanExporter - Registering beans for JMX exposure on startup
14:23:29.978 [main] INFO  o.s.c.s.DefaultLifecycleProcessor - Starting beans in phase 0
14:23:30.103 [main] INFO  c.p.spring.cloud.task.TaskListener - Task Startup
14:23:30.109 [main] INFO  c.p.spring.cloud.task.TaskDemo - Hello World from Spring Cloud Task!
14:23:30.113 [main] INFO  c.p.spring.cloud.task.TaskListener - End of Task
14:23:30.126 [main] INFO  o.s.c.a.AnnotationConfigApplicationContext - Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@2a798d51: startup date [Fri Oct 12 14:23:28 CST 2018]; root of context hierarchy
14:23:30.127 [main] INFO  o.s.c.s.DefaultLifecycleProcessor - Stopping beans in phase 0
14:23:30.128 [main] INFO  o.s.j.e.a.AnnotationMBeanExporter - Unregistering JMX-exposed beans on shutdown
14:23:30.128 [main] INFO  o.s.o.j.LocalContainerEntityManagerFactoryBean - Closing JPA EntityManagerFactory for persistence unit 'default'
14:23:30.129 [main] INFO  o.h.tool.hbm2ddl.SchemaExport - HHH000227: Running hbm2ddl schema export
14:23:30.129 [main] INFO  o.h.tool.hbm2ddl.SchemaExport - HHH000230: Schema export complete
14:23:30.132 [main] INFO  c.p.spring.cloud.task.TaskDemo - Started TaskDemo in 2.405 seconds (JVM running for 2.771)
```

# 4.  与Spring Batch集成
我们可以将Spring Batch Job作为Task执行，并使用Spring Cloud Task记录Job执行的事件。要启用此功能，我们需要添加与Boot和Cloud相关的Batch依赖项：
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-task-batch</artifactId>
</dependency>
```
要将作业配置为任务，我们需要在JobConfiguration类中注册Job Bean ：
```
@Bean
	public Job job2() {
		return jobBuilderFactory.get("job2").start(stepBuilderFactory.get("job2step1").tasklet(new Tasklet() {
			@Override
			public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
				LOGGER.info("This job is from PeterWanghao");
				return RepeatStatus.FINISHED;
			}
		}).build()).build();
	}
```
我们需要使用@EnableBatchProcessing注解来装饰TaskDemo类：
```
@SpringBootApplication
@EnableTask
@EnableBatchProcessing
public class TaskDemo {
    // ...
}
```
该@EnableBatchProcessing注解集成了Spring Batch的功能，并设置批处理作业所需的基本配置。

现在，如果我们运行应用程序，@ EnableBatchProcessing注释将触发Spring Batch Job执行，Spring Cloud Task将使用springcloud数据库记录所有批处理作业的执行事件。

运行TaskDemo类，在控制台可看到任务被执行：
```
14:30:26.003 [main] INFO  o.s.j.e.a.AnnotationMBeanExporter - Registering beans for JMX exposure on startup
14:30:26.008 [main] INFO  o.s.c.s.DefaultLifecycleProcessor - Starting beans in phase 0
14:30:26.047 [main] INFO  c.p.spring.cloud.task.TaskListener - Task Startup
14:30:26.053 [main] INFO  c.p.spring.cloud.task.TaskDemo - Hello World from Spring Cloud Task!
14:30:26.054 [main] INFO  o.s.b.a.b.JobLauncherCommandLineRunner - Running default command line with: []
14:30:26.257 [main] INFO  o.s.b.c.l.support.SimpleJobLauncher - Job: [SimpleJob: [name=job1]] launched with the following parameters: [{}]
14:30:26.266 [main] INFO  o.s.c.t.b.l.TaskBatchExecutionListener - The job execution id 1 was run within the task execution 4
14:30:26.292 [main] INFO  o.s.batch.core.job.SimpleStepHandler - Executing step: [job1step1]
14:30:26.312 [main] INFO  c.p.s.cloud.task.JobConfiguration - Tasklet has run
14:30:26.342 [main] INFO  o.s.batch.core.job.SimpleStepHandler - Executing step: [job1step2]
14:30:26.353 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.353 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.353 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.354 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -7
14:30:26.354 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -2
14:30:26.354 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -3
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - Processing of chunks
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -10
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -5
14:30:26.359 [main] INFO  c.p.s.cloud.task.JobConfiguration - >> -6
14:30:26.381 [main] INFO  o.s.b.c.l.support.SimpleJobLauncher - Job: [SimpleJob: [name=job1]] completed with the following parameters: [{}] and the following status: [COMPLETED]
14:30:26.398 [main] INFO  o.s.b.c.l.support.SimpleJobLauncher - Job: [SimpleJob: [name=job2]] launched with the following parameters: [{}]
14:30:26.404 [main] INFO  o.s.c.t.b.l.TaskBatchExecutionListener - The job execution id 2 was run within the task execution 4
14:30:26.420 [main] INFO  o.s.batch.core.job.SimpleStepHandler - Executing step: [job2step1]
14:30:26.428 [main] INFO  c.p.s.cloud.task.JobConfiguration - This job is from PeterWanghao
14:30:26.441 [main] INFO  o.s.b.c.l.support.SimpleJobLauncher - Job: [SimpleJob: [name=job2]] completed with the following parameters: [{}] and the following status: [COMPLETED]
14:30:26.442 [main] INFO  c.p.spring.cloud.task.TaskListener - End of Task
14:30:26.448 [main] INFO  o.s.c.a.AnnotationConfigApplicationContext - Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@399f45b1: startup date [Fri Oct 12 14:30:23 CST 2018]; root of context hierarchy
14:30:26.450 [main] INFO  o.s.c.s.DefaultLifecycleProcessor - Stopping beans in phase 0
14:30:26.450 [main] INFO  o.s.j.e.a.AnnotationMBeanExporter - Unregistering JMX-exposed beans on shutdown
14:30:26.451 [main] INFO  o.s.o.j.LocalContainerEntityManagerFactoryBean - Closing JPA EntityManagerFactory for persistence unit 'default'
14:30:26.451 [main] INFO  o.h.tool.hbm2ddl.SchemaExport - HHH000227: Running hbm2ddl schema export
14:30:26.451 [main] INFO  o.h.tool.hbm2ddl.SchemaExport - HHH000230: Schema export complete
14:30:26.455 [main] INFO  c.p.spring.cloud.task.TaskDemo - Started TaskDemo in 3.746 seconds (JVM running for 4.093)
```

# 5.  从Stream启动任务
我们可以从Spring Cloud Stream触发任务。为了达到这个目的，我们使用@EnableTaskLaucnher注解。这一次，我们使用Spring Boot应用程序添加注释，TaskSink将可用：
```
@SpringBootApplication
@EnableTaskLauncher
public class StreamTaskSinkApplication {
    public static void main(String[] args) {
        SpringApplication.run(TaskSinkApplication.class, args);
    }
}
```
TaskSink从一个流中接收消息，信息中包含GenericMessage含有TaskLaunchRequest作为有效负载。然后它触发任务启动请求中提供的基于任务的坐标。

要使TaskSink起作用，我们需要配置一个实现TaskLauncher接口的bean。出于测试目的，我们在这里Mock实现：
```
@Bean
public TaskLauncher taskLauncher() {
    return mock(TaskLauncher.class);
}
```
我们需要注意的是，TaskLauncher接口仅在添加spring-cloud-deployer-local依赖项后才可用：
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-deployer-local</artifactId>
    <version>1.3.1.RELEASE</version>
</dependency>
```
我们可以测试通过调用发起的任务是否输入了Sink：
```
public class SpringCloudTaskSinkApplicationIntegrationTest{
    
    @Autowired
    private Sink sink; 
     
    //
}
```
现在，我们创建一个TaskLaunchRequest实例，并将其作为GenericMessage < TaskLaunchRequest >对象的有效负载发送。然后我们可以调用Sink的输入通道，将GenericMessage对象保留在通道中。

# 6. 结论
在本文中，我们探讨了Spring Cloud Task的执行方式以及如何配置它以在数据库中记录其事件。我们还观察了如何定义Spring Batch作业并将其存储在TaskRepository中。最后，我们解释了如何从Spring Cloud Stream中触发Task。
