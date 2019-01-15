# 第五节 外部化配置Archaius

## 1.概述

Netflix [Archaius](https://github.com/Netflix/archaius)  是一个功能强大的配置管理库。它是一个可用于从许多不同来源收集配置属性的框架，提供对配置信息的快速及线程安全访问。

除此之外，Archaius允许属性在运行时动态更改，使系统无需重新启动应用程序即可获得这些变化。

在这个介绍性文章中，我们将设置一个简单的Spring Cloud Archaius配置，我们将解释底层发生了什么，最后，我们将看到Spring如何扩展基本设置。

## 2.Netflix Archaius功能

众所周知，Spring Boot已经提供了管理外部化配置的工具，为什么还要设置不同的机制呢？

因为Archaius提供了一些其他任何配置框架都没有考虑过的方便有趣的功能。其中的一些关键点是：

 - 动态和类型属性   
 - 在属性改变时调用的回调机制   
 - 动态配置源（如URL，JDBC和Amazon DynamoDB）的实现   
 - Spring Boot Actuator或JConsole可以访问的JMX MBean，用于检查和操作属性   
 - 动态属性验证

因此，Spring Cloud已经开发了一个库，可以轻松配置“Spring Environment Bridge”，以便Archaius可以从Spring Environment中读取属性。

## 3.依赖性

让我们将  spring-cloud-starter-netflix-archaius 添加到我们的应用程序中，它将为我们的项目添加所有必要的依赖项。

或者，我们也可以将spring-cloud-netflix添加到我们的  dependencyManagement 部分，并依赖于其工件版本的规范：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-archaius</artifactId>
</dependency>
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloudgroupId>
			<artifactId>spring-cloud-netflixartifactId>
			<version>2.0.1.RELEASEversion>
			<type>pomtype>
			<scope>importscope>
		<dependency>
	<dependencies>
<dependencyManagement>
```

## 4.用法

一旦我们添加了所需的依赖项，我们就能够访问框架管理的属性：
```
DynamicStringProperty propertyOneWithDynamic = DynamicPropertyFactory.getInstance()
			.getStringProperty("springcloud.archaius.properties.one", "not found!");
String propertyCurrentValue = dynamicProperty.get();
```

让我们以一个简短的例子来看看它是如何开箱即用的。

### 4.1 快速示例

默认情况下，它动态管理应用程序类路径中名为config.properties的文件中定义的所有属性。

所以我们将它添加到我们的资源文件夹中，其中包含一些任意属性：
```
springcloud.archaius.properties.one=one FROM:config.properties
springcloud.archaius.properties.three=three FROM:config.properties
```

现在我们需要一种在任何特定时刻检查属性值的方法。在这种情况下，我们将创建一个RestController，将值作为JSON响应检索：
```
@RestController
public class ConfigPropertiesController {
	private DynamicStringProperty propertyOneWithDynamic = DynamicPropertyFactory.getInstance()
			.getStringProperty("springcloud.archaius.properties.one", "not found!");

	private DynamicStringProperty propertyTwoWithDynamic = DynamicPropertyFactory.getInstance()
			.getStringProperty("springcloud.archaius.properties.two", "not found!");

	private DynamicStringProperty propertyThreeWithDynamic = DynamicPropertyFactory.getInstance()
			.getStringProperty("springcloud.archaius.properties.three", "not found!");

	private DynamicStringProperty propertyFourWithDynamic = DynamicPropertyFactory.getInstance()
			.getStringProperty("springcloud.archaius.properties.four", "not found!");

	@GetMapping("/properties-from-dynamic")
	public Map<String, String> getPropertiesFromDynamic() {
		Map<String, String> properties = new HashMap<>();
		properties.put(propertyOneWithDynamic.getName(), propertyOneWithDynamic.get());
		properties.put(propertyTwoWithDynamic.getName(), propertyTwoWithDynamic.get());
		properties.put(propertyThreeWithDynamic.getName(), propertyThreeWithDynamic.get());
		properties.put(propertyFourWithDynamic.getName(), propertyFourWithDynamic.get());
		return properties;
	}
}
```
我们来试试吧。我们可以向/properties-from-dynamic发送请求，服务将按预期检索存储在config.properties中 的值。

到目前为止没什么大不了的，对吧？好的，让我们继续并更改类路径文件中属性的值，而无需重新启动服务。在一分钟左右之后，对端点的调用应检索出新值。

接下来，我们将尝试了解幕后发生的事情。

## 5.它是如何工作的？

首先，让我们试着理解大局。

Archaius是Apache的[Commons Configuration](http://commons.apache.org/proper/commons-configuration/)库的扩展，添加了一些很好的功能，如动态源的轮询框架，具有高吞吐量和线程安全的实现。

然后  spring-cloud-netflix-archaius  库进入，合并所有不同的属性源，并使用这些源自动配置Archaius工具。

### 5.1 Netflix Archaius库

它定义了一个复合配置，是可以从不同来源获得的各种配置的集合。

此外，其中一些配置源可以支持在运行时轮询更改。Archaius提供接口和一些预定义的实现来配置这些类型的源。

源集合是分层的，因此如果属性存在于多个配置中，则最终值将是最顶部插槽中的值。

最后，  ConfigurationManager处理系统范围的配置和部署上下文。它可以安装最终的复合配置，或检索已安装的复合配置进行修改。

### 5.2 Spring Cloud支持

Spring Cloud Archaius库的主要任务是将所有不同的配置源合并为  ConcurrentCompositeConfiguration，并使用ConfigurationManager进行安装  。

库定义源的优先顺序是：
    1. 上下文中定义的任何Apache公共配置AbstractConfiguration bean
    2. Autowired  Spring ConfigurableEnvironment中定义的所有源代码
    3. 默认的Archaius源，我们在上面的例子中看到过
    4. Apache的SystemConfiguration和EnvironmentConfiguration  源

Spring Cloud库提供的另一个有用功能是定义一个Actuator Endpoint 来监控属性并与之交互。

## 6.调整和扩展Archaius配置

现在我们已经更好地理解了Archaius的工作原理，我们很好地分析了如何使配置适应我们的应用程序，或者如何使用我们的配置源扩展功能。

### 6.1 Archaius支持的配置属性

如果我们希望Archaius考虑类似于config.properties的其他配置文件 ，我们可以定义  archaius.configurationSource.additionalUrls系统属性。

该值被解析为由逗号分隔的URL列表，因此，例如，我们可以在启动应用程序时添加此系统属性：
```
-Darchaius.configurationSource.additionalUrls="classpath:other-dir/extra.properties,file:///home/user/other-extra.properties"
```
Archaius将首先读取config.properties文件，然后按指定的顺序读取其他文件。因此，后面文件中定义的属性将优先于先前的属性。

我们可以使用几个其他系统属性来配置Archaius默认配置的各个方面：


 - archaius.configurationSource.defaultFileName：类路径中的默认配置文件名   
 - archaius.fixedDelayPollingScheduler.initialDelayMills：读取配置源之前的初始延迟  
 - archaius.fixedDelayPollingScheduler.delayMills：两次读取源之间的延迟; 默认值为1分钟

### 6.2 使用Spring添加其他配置源

我们如何添加一个不同的配置源来由所描述的框架管理？我们如何管理优先级高于Spring环境中定义的动态属性？

回顾我们在4.2节中提到的内容，我们可以发现Spring定义的Composite Configuration中的最高配置是在上下文中定义的  AbstractConfiguration bean。

因此，我们需要做的就是使用Archaius提供的一些功能将这个Apache的抽象类的实现添加到我们的Spring Context中，Spring的自动配置会自动将它添加到托管配置属性中。

为了简单起见，我们将看到一个示例，我们配置一个类似于默认config.properties的属性文件，但其优先级高于Spring环境和应用程序属性的其余部分：
```
@Configuration
public class ApplicationPropertiesConfigurations {
	@Bean
	public AbstractConfiguration addApplicationPropertiesSource() {
		PolledConfigurationSource source = new URLConfigurationSource("classpath:other-config.properties");
		return new DynamicConfiguration(source, new FixedDelayPollingScheduler());
	}
}
```
幸运的是，它考虑了几个配置源，我们几乎可以毫不费力地设置。

## 7.结论

总而言之，我们已经了解了Archaius以及它为利用配置管理提供的一些很酷的功能。

此外，我们还看到了Spring Cloud自动配置库如何发挥作用，使我们可以方便地使用该库的API。