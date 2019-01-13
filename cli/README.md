# 第九章 Spring Cloud CLI

## 1.简介

在本节中，我们将介绍Spring Boot Cloud CLI（或简称Cloud CLI）。该工具为Spring Boot CLI提供了一组命令行增强功能，有助于进一步抽象和简化Spring Cloud部署。

CLI于2016年底推出，允许使用命令行、.yml配置文件和Groovy脚本快速自动配置和部署标准Spring Cloud服务。

## 2.安装

Spring Boot Cloud CLI 1.3.x需要Spring Boot CLI 1.5.x，因此请务必从[Maven Central](https://search.maven.org/classic/#search%7Cga%7C1%7Ca%3A%22spring-boot-cli%22)获取最新版本的Spring Boot CLI （[安装说明](https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started-installing-spring-boot.html#getting-started-manual-cli-installation)）以及[Maven资源库](https://search.maven.org/classic/#search%7Cga%7C1%7Ca%3A%22spring-cloud-cli%22)中的最新版本的Cloud CLI （[官方Spring存储库](https://repo.spring.io/snapshot/org/springframework/cloud/spring-cloud-cli/)）！

本文中使用的是spring-boot-cli 1.5.18.RELEASE，spring-cloud-cli 1.3.2.RELEASE。

1、首先安装Spring Boot CLI并可以使用，验证只需运行：
```
$ spring --version
```

2、然后安装最新的稳定版Cloud CLI：
```
$ spring installorg.springframework.cloud:spring-cloud-cli:1.3.2.RELEASE
```

3、最后验证Cloud CLI：
```
$ spring cloud --version
```

可以在[官方Cloud CLI 页面](https://cloud.spring.io/spring-cloud-cli/)上找到更多安装说明！

## 3.默认服务和配置

CLI提供七种核心服务，可以使用单行命令运行和部署。

在http://localhost:8888上启动Cloud Config服务器：
```
$ spring cloud configserver
```

http://localhost:8761上启动Eureka服务器：
```
$ spring cloud eureka
```

在http://localhost:9095上启动H2服务器：
```
1 $ spring cloud h2
```

在http://localhost:9091上启动Kafka服务器：
```
$ spring cloud kafka
```

在http://localhost:9411上启动Zipkin服务器：
```
$ spring cloud zipkin
```

在http://localhost:9393上启动Dataflow服务器：
```
$ spring cloud dataflow
```

在http://localhost:7979上启动Hystrix仪表板：
```
$ spring cloud hystrixdashboard
```

列出可部署的应用服务：
```
$ spring cloud --list
```

帮助命令：
```
$ spring help cloud
```

有关这些命令的更多详细信息，请查看官方[博客](https://spring.io/blog/2016/11/02/introducing-the-spring-cloud-cli-launcher)。

## 4.使用YML自定义云服务

还可以使用相应命名的.yml文件配置可通过Cloud CLI部署的云服务：
```
spring:
  profiles:
    active: git
  cloud:
    config:
      server:
        git:
          uri: https://github.com/spring-cloud-samples/config-repo
```

这是一个简单的配置文件，我们可以使用它来启动Cloud Config Server。

上面的例子中，我们指定一个Git存储库作为URI源，当我们发出'spring cloud configserver'命令时，它将被自动克隆和部署。

Cloud CLI使用Spring Cloud Launcher。这意味着Cloud CLI支持大多数Spring Boot配置机制。这是Spring Boot属性的[官方列表](https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html)。

Spring Cloud配置符合'spring.cloud ... '约定。可以在此[链接](http://cloud.spring.io/spring-cloud-static/spring-cloud-config/1.3.3.RELEASE/single/spring-cloud-config.html#_environment_repository)中找到Spring Cloud和Spring Config Server的设置。

我们还可以直接在下面的yml文件中指定几个不同的模块和服务：
```
spring:
  cloud:
    launcher:
      deployables:
        - name: configserver
          coordinates: maven://...:spring-cloud-launcher-configserver:1.3.2.RELEASE
          port: 8888
          waitUntilStarted: true
          order: -10
        - name: eureka
          coordinates: maven:/...:spring-cloud-launcher-eureka:1.3.2.RELEASE
          port: 8761
```

该yml允许通过使用Maven或Git库，添加自定义的服务或模块。

## 5.运行自定义Groovy脚本

自定义组件可以用Groovy编写并高效部署，因为Cloud CLI可以编译和部署Groovy代码。

这是一个非常简单的REST API实现示例：
```
@RestController
@RequestMapping('/api')
class api {
    @GetMapping('/get')
    def get() { [message: 'Hello'] }
}
```

假设脚本保存为restapi.groovy，我们可以像这样启动我们的服务：
```
$ spring run restapi.groovy
```

访问http://localhost:8080/api/get应该显示：
```
{"message":"Hello"}
```

## 6.加密/解密

Cloud CLI还提供了加密和解密工具（可在org.springframework.cloud.cli.command.*中找到），可以直接通过命令行使用，也可以通过将值传递给Cloud Config Server端点来间接使用。

让我们设置它，看看如何使用它。

### 6.1 安装

Cloud CLI和Spring Cloud Config Server都使用org.springframework.security.crypto.encrypt.* 来处理加密和解密的命令。

因此，都需要通过Oracle提供的JCE Unlimited Strength Extension，可在[这里](http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html)找到。

### 6.2 命令行加密和解密

要通过终端加密' my_value '，请调用：
```
$ spring encrypt my_value --key my_key
```

可以使用“@”后跟路径（通常用于RSA公钥）代表文件路径（替换上面的“ my_key ”）：
```
$ spring encrypt my_value --key @${WORKSPACE}/foos/foo.pub
```

'my_value'现在将被加密为：
```
c93cb36ce1d09d7d62dffd156ef742faaa56f97f135ebd05e90355f80290ce6b
```

可通过命令行进行解密：
```
$ spring decrypt --key my_key c93cb36ce1d09d7d62dffd156ef742faaa56f97f135ebd05e90355f80290ce6b
```

我们现在还可以在配置YAML或属性文件中使用加密值，在加载时它将由Cloud Config Server自动解密：
```
encrypted_credential: "{cipher}c93cb36ce1d09d7d62dffd156ef742faaa56f97f135ebd05e90355f80290ce6b"
```

### 6.3 使用Config Server加密和解密

Spring Cloud Config Server公开RESTful端点，其中密钥和加密值对可以存储在Java安全存储或内存中。

有关如何正确设置和配置Cloud Config Server以接受对称或非对称加密的详细信息，请查看[官方文档](http://cloud.spring.io/spring-cloud-static/spring-cloud-config/1.3.3.RELEASE/single/spring-cloud-config.html#_encryption_and_decryption)。

使用“ spring cloud configserver ”命令配置并运行Spring Cloud Config Server后，您就可以调用其API：
```
$ curl localhost:8888/encrypt-d mysecret
//682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda$ 
curl localhost:8888/decrypt-d 682bc583f4641835fa2db009355293665d2647dade3375c0ee201de2a49f7bda
//mysecret
```

## 7.结论

我们在这里专注于Spring Boot Cloud CLI的介绍。有关更多的信息，请查看[官方文档](http://cloud.spring.io/spring-cloud-static/spring-cloud-cli/1.3.2.RELEASE/)。