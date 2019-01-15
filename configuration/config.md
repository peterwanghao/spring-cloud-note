# 第一节 Spring Cloud Config

Spring Cloud Config是Spring创建的一个全新的服务端/客户端项目，为应用提供分布式的配置存储，提供集中化的外部配置支持。它除了适用于Spring构建的应用程序外，也可以在其他语言运行的应用程序中使用。

Spring Cloud Config分为服务端和客户端两部分。其中服务端称为分布式配置中心，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息的功能；客户端则是微服务架构中的各个微服务应用，它们通过指定的配置中心来管理与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息。

Spring Cloud Config实现的配置中心默认采用Git来存储配置信息，但也提供了对其他存储方式的支持，比如SVN仓库、本地文件系统。在本例中构建了一个基于Git存储的分布式配置中心，并在客户端中演示了如何制定应用所属的配置中心，并能够从配置中心获取配置信息的整个过程。

## 1 准备Git服务器

在Windows系统下搭建本地的Git服务器，可参看教程  [http://blog.csdn.net/qwer971211/article/details/71156055](http://blog.csdn.net/qwer971211/article/details/71156055)

创建一个新的版本库
![1.png](https://upload-images.jianshu.io/upload_images/11110195-e5242d16cba9bd0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在此版本库里创建config-repo目录，提交两个配置文件
config-client-development.properties 
config-client-production.properties
```
user.role=Developer
user.password=pass
```

```
user.role=User
user.password=pass
```

##2 服务端（配置中心）

### 2.1 项目依赖

创建一个Maven工程，引入spring-cloud-config-server模块依赖
```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

### 2.2 服务端应用实现

一个SpringBoot应用通过自动配置注释@EnableConfigServer引入所有必需的设置。
```
@SpringBootApplication
@EnableConfigServer
public class ConfigServer {
	public static void main(String[] args) {
		SpringApplication.run(ConfigServer.class, args);
	}
}
```

### 2.3 服务端配置
配置Git仓库信息和Basic-Authentication的用户名和密码
```
spring.application.name=config-server
server.port=8888

#配置Git仓库位置
spring.cloud.config.server.git.uri=ssh://admin@localhost:29418/Test.git
#配置仓库路径下的相对搜索位置，可以配置多个
spring.cloud.config.server.git.searchPaths=config-repo
#配置为true表示启动时就克隆配置缓存到本地。
spring.cloud.config.server.git.clone-on-start=true
#访问Git仓库的用户名
spring.cloud.config.server.git.username=admin
#访问Git仓库的用户密码
spring.cloud.config.server.git.password=admin

#Basic-Authentication
security.user.name=root
security.user.password=s3cr3t
```

### 2.4 服务端运行
访问URL规则如下，其中{label}占位符为引用Git分支，{application}为引用客户端的应用程序名称，{profile}为引用客户端当前活动的应用程序配置文件。
```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```
启动工程，访问 [http://localhost:8888/config-client/development/master](http://localhost:8888/config-client/development/master)
得到配置信息，证明服务端已配置正确。
```
{
  "name": "config-client",
  "profiles": [
    "development"
  ],
  "label": "master",
  "version": "45bc95dec293023587669efb86cef8be61d1c1d7",
  "state": null,
  "propertySources": [
    {
      "name": "ssh://admin@localhost:29418/Test.git/config-repo/config-client-development.properties",
      "source": {
        "user.password": "pass",
        "user.role": "Developer"
      }
    }
  ]
}
```
## 3 客户端（微服务应用）

### 3.1 项目依赖
```
        <dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
```

### 3.2 客户端应用实现
```
@SpringBootApplication
@RestController
public class ConfigClient {
	@Value("${user.role}")
	private String role;

	@Value("${user.password}")
	private String password;

	public static void main(String[] args) {
		SpringApplication.run(ConfigClient.class, args);
	}

	@RequestMapping(value = "/whoami/{username}", method = RequestMethod.GET, produces = MediaType.TEXT_PLAIN_VALUE)
	public String whoami(@PathVariable("username") String username) {
		return String.format("Hello %s! You are a(n) %s and your password is '%s'.\n", username, role, password);
	}
}
```
### 3.3 客户端配置
在bootstrap.properties文件中配置
```
spring.application.name=config-client
server.port=8000

spring.cloud.config.uri=http://localhost:8888
spring.cloud.config.profile=development
spring.cloud.config.label=master
spring.cloud.config.username=root
spring.cloud.config.password=s3cr3t
spring.cloud.config.fail-fast=true
```
### 3.4 客户端运行

[http://localhost:8000/whoami/Peter](http://localhost:8000/whoami/Peter)

返回 Hello Peter! You are a(n) Developer and your password is 'pass'.

## 4 配置项加密/解密

在使用Spring Cloud Config的加密解密功能时，有一个必要的前提需要我们注意。为了启动该功能，我们需要在配置中心的运行环境中安装不限长度的JCE版本（Unlimited Strength Java Cryptography Extension）。虽然，JCE功能在JRE中自带，但是默认使用的是有长度限制的版本。我们可以从Oracle的官方网站下载到它，它是一个压缩包，解压后可以看到下面三个文件：

- README.txt
- Local_policy.jar
- US_export_policy.jar

我们需要将local_policy.jar和US_export_policy.jar两个文件复制$JAVA_HOME/jre/lib/security目录下，覆盖原来的默认内容。到这里，加密解密的准备工作就完成了。

### 4.1 对称加密

通过encrypt.key属性在配置文件bootstrap.properties中直接指定密钥信息（对称性密钥）。
```
encrypt.key=symmetricKey
```

通过POST 使用/encrypt对请求的body内容进行加密，得到密文
```
$ curl localhost:8888/encrypt -d mysecret
```

保存到配置文件提交到Git仓库
```
user.role=Developer
user.password={cipher}4e9908d0197d510a2250e5cca64991f525b56cc0ec0983d23c87531ba8d9959c

```

调用[http://localhost:8000/whoami/Peter](http://localhost:8000/whoami/Peter) 可看到我们的配置值被正确解密

Hello Peter! You are a(n) Developer and your password is 'mysecret'.

### 4.2 非对称加密

首先使用Java keytool生成一个新的密钥库，包括一个RSA密钥对
```
$> keytool -genkeypair -alias config-server-key \
       -keyalg RSA -keysize 4096 -sigalg SHA512withRSA \
       -dname 'CN=Config Server,OU=Spring Cloud,O=Baeldung' \
       -keypass my-k34-s3cr3t -keystore config-server.jks \
       -storepass my-s70r3-s3cr3t
```
之后，我们将创建的密钥库添加到我们的服务器的bootstrap.properties中并重新运行它
```
#非对称密钥
encrypt.key-store.location=classpath:/config-server.jks
encrypt.key-store.password=my-s70r3-s3cr3t
encrypt.key-store.alias=config-server-key
encrypt.key-store.secret=my-k34-s3cr3t
```
同理通过POST 使用/encrypt对请求的body内容进行加密，得到密文
```
$ curl localhost:8888/encrypt -d mysecret
```
保存到配置文件提交到Git仓库
```
user.role=Developer
user.password={cipher}AgAN74/iWyGPPzFNSKNJTuTvqgLlKZA0OW2EkBTdSuJKrZ4QAx3czXctQHJCY5SHO17wymGFLRtHZa23Hw8TPovHnymEW+WC6qus/aVWt4WFR+QK3LTNu9/+jZv+2rZff0eHGRcwWus43BIvyFdP/LjnO0M68WcIE26KZmlO2WnLCmAaw3rtdG1M4kJnCQrneG9cVR2xg/fjM4QyTAOM/oV8xNani5yH1oDza5JIaDmJMzkjtvo04vZ6PxdhuRlPDjygJ/CxTyjHd+bZqDBPfSH1IkOSv6tMg2G5o54Bch1nFqxMvPLi9sF5WaKpJRqLlPkua/Jz4Jat3NgsQ+fSgWdNZX0ukC+TT067wOffACZdYAd6mwBUvk97QKQCBR4htInTD7KKu84D8iHQHw41TS5sJds98AESvFtSc7+2jI8i0V1eEsM56ohoATC9KYpOuS0d3zydkrB3n3VNVFijbeHQjoYpQqZIXvuFSCwp0Aj1xPR/R7zdvOMS0t0uH4xavk0a194RQ2XRpAZN+g5+wdmQsqTH4OVgUWK/Jqr3nMWbGevlKDQ3e8gXV3S362feWoB78Y3ZHLxeM9ZSaZn2yqKewbzqbnrWDE7MtYjmslZdDM+qQ/jN2+IBHcIpQcWzwZiWYpPp/vJZ9xqy3DoNIzH5UrP8L9xDhpOggfxxgnLw1+PHqO88yKJfbulWpt6A5CwmRC+TBXLbbWsWc06JdlCh

```
调用[http://localhost:8000/whoami/Peter](http://localhost:8000/whoami/Peter) 可看到我们的配置值被正确解密

Hello Peter! You are a(n) Developer and your password is 'mysecret'.

## 5 总结
现在我们已经可以创建一个配置服务器来提供从Git存储库到客户端应用程序的一组操作的实现。我们也可以将Config Server作为一个普通的微服务应用，纳入到Eureka的服务治理体系中。对于服务端的负载均衡配置和客户端的配置中心指定都通过服务治理机制一并解决，即实现了高可用，也可实现自我维护。