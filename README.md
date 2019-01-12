# 本书内容

本书介绍的是微服务框架Spring Cloud，整理了在学习Spring Cloud过程中所记录的笔记。Spring Cloud包含了非常多的子框架，书中每一个章节分别针对其中一个子框架进行说明并辅以一个实际的例子。本书希望能对Spring Cloud的初学者予以一定帮助。



第一章，是Spring Cloud的基础综述，介绍了Spring Cloud的组成、特征以及它是如何定义版本的。



第二章，是Spring Cloud的核心组件Spring Cloud Netflix。对多个Netflix OSS开源套件进行了整合。 



第三章，配置管理工具。Spring Cloud Config支持使用git存储配置内容，可以使用它实现应用配置的外部化存储，并支持客户端配置信息刷新，加密、解密配置内容等。Vault是安全访问私密数据的工具，可以作为Spring Cloud Config服务器后端。



第四章，基于Consul，Zookeeper的服务发现和配置管理组件。



第五章，服务跟踪。Spring Cloud Sleuth是Spring Cloud 应用的分布式跟踪实现，可以完美整合ZipKin。



第六章，服务安全。Spring Cloud Security是基于Spring Security的安全工具包，为应用程序添加安全控制。提供在Zuul代理中对OAuth2客户端请求的中继器。



第七章，Spring Cloud Stream通过RabbitMQ或者Kafka实现消费微服务，可以通过简单的声明式模型来发送和接受消息。



第八章，Spring Cloud Task的目标是为Spring Boot应用程序提供创建短运行期微服务的功能。



第九章，Spring Cloud CLI。用于在groovy中快速创建Spring Cloud 应用的Spring Boot CLI插件。



