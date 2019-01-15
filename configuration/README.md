# 第三章 配置管理

Spring Cloud Config是Spring Cloud团队创建的一个全新项目，用来为分布式系统中的基础设施和微服务应用提供集中化的外部配置支持，它分为服务端与客户端两个部分。其中服务端也称为分布式配置中心，它是一个独立的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息等访问接口；而客户端则是微服务架构中的各个微服务应用或基础设施，它们通过指定的配置中心来管理应用资源与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息。



分布式配置中心的基本结构：



- 远程Git仓库：用来存储配置文件的地方。
- Config Server：分布式配置中心config-server工程。在该工程中指定了所要连接的Git仓库位置以及账户、密码等连接信息。
- 本地Git仓库：在Config Server的文件系统中，每次客户端请求获取配置信息时，Config Server从Git仓库中获取最新配置到本地，然后在本地Git仓库中读取并返回。当远程仓库无法获取时，直接将本地内容返回。
- Service A、Service B：具体的微服务应用，它们指定了Config Server的地址，从而实现从外部化获取应用自己要用的配置信息。这些应用在启动的时候，会向Config Server请求获取配置信息来进行加载。

 

**配置中心**

引入依赖 artifactId : spring-cloud-config-server

添加@EnableConfigServer注解

在application.properties中添加服务的基本信息以及Git仓库的相关信息

 

**配置信息加密/解密**

在使用Spring Cloud Config的加密解密功能时，有一个必要的前提需要我们注意。为了启动该功能，我们需要在配置中心的运行环境中安装不限长度的JCE版本（Unlimited Strength Java Cryptography Extension）。虽然，JCE功能在JRE中自带，但是默认使用的是有长度限制的版本。我们可以从Oracle的官方网站下载到它，它是一个压缩包，解压后可以看到下面三个文件：

&emsp;&emsp;README.txt

&emsp;&emsp;Local_policy.jar

&emsp;&emsp;US_export_policy.jar

我们需要将local_policy.jar和US_export_policy.jar两个文件复制到$JAVA_HOME/jre/lib/security目录下，覆盖原来的默认内容。