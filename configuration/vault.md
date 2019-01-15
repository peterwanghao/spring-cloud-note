# 第二节 Spring Cloud Vault

## 1.概述
在本文中我们将展示如何在Spring Boot应用程序中使用Hashicorp的Vault来保护敏感的配置数据。

我们在这里假设你已经掌握了一些Vault知识，并且已经准备好了运行环境。如果不是这样的话，让我们花点时间阅读下之前的文章[Vault 介绍教程](https://blog.csdn.net/peterwanghao/article/details/83181932)，以便我们熟悉其基础知识。

## 2. Spring Cloud Vault
Spring Cloud Vault是Spring Cloud堆栈的一个新增功能，允许应用程序以透明的方式访问存储在Vault实例中的机密。

通常，迁移到Vault是一个非常简单的过程：只需添加所需的库并为项目添加一些额外的配置属性，我们应该很高兴。无需更改代码！

这是可能的，因为它充当在当前环境中注册的高优先级PropertySource。因此，只需要属性，Spring就会使用它。示例包括DataSource属性，ConfigurationProperties等。

## 3.将Spring Cloud Vault添加到Spring Boot项目
为了在基于Maven的Spring Boot项目中包含spring-cloud-vault库，我们使用相关的启动工件，它将获取所有必需的依赖项。

除了主要启动器之外，我们还将包括spring-vault-config-databases，它增加了对动态数据库凭据的支持：
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-vault-config-databases</artifactId>
</dependency>
```

**基本配置**

为了正常工作，Spring Cloud Vault需要一种方法来确定联系Vault服务器的位置以及如何针对它进行身份验证。

我们通过在bootstrap.yml或bootstrap.properties中提供必要的信息来实现此目的：
```
spring:
  cloud:
    vault:
      uri: https://localhost:8200
      connection-timeout: 5000
      read-timeout: 15000
      config:
          order: -10
          
      ssl:
        trust-store: classpath:/vault.jks
        trust-store-password: changeit
```
该spring.cloud.vault.uri属性指向Vault库的API地址。由于我们的测试环境使用带有自签名证书的HTTPS，因此我们还需要提供包含其公钥的密钥库。

请注意，此配置没有身份验证数据。对于最简单的情况，我们使用固定令牌，我们可以通过系统属性spring.cloud.vault.token或环境变量传递它。这种方法可以与标准云配置机制结合使用，例如Kubernetes的ConfigMaps或Docker机密。

Spring Vault还需要为我们要在应用程序中使用的每种类型的Secret配置额外的配置。以下部分描述了我们如何添加对两种常见Secret类型的支持：键/值和数据库凭据。
```
kv:
  enabled: false
  backend: kv
  application-name: fakebank
  
database:
  enabled: true
  role: fakebank-accounts-ro
  backend: database
  username-property: spring.datasource.username
  password-property: spring.datasource.password
```
## 4.使用Generic Secrets后端
我们使用通用机密后端来访问存储在Vault中的键值对的无版权机密。

假设我们的类路径中已经有spring-cloud-starter-vault-config依赖项，我们所要做的就是在应用程序的bootstrap.yml配置文件中添加一些属性：
```
spring:
  cloud:
    vault:
      generic:
        enabled: true
        application-name: fakebank
```
在这种情况下，属性application-name是可选的。如果未指定，Spring将采用标准spring.application.name的值。

我们现在可以使用存储在secret / fakebank中的所有键/值对作为任何其他Environment属性。以下代码段显示了我们如何读取存储在此路径下的api_key键的值：
```
@Autowired Environment env;
public String getApiKey() {
    return env.getProperty("api_key");
}
```
我们可以看到，代码本身对Vault一无所知，这是一件好事！我们仍然可以在本地测试中使用固定属性，并通过在bootstrap.yml中启用单个属性来切换到Vault 。

**关于Spring配置文件的一点说明**

如果在当前环境中可用，则Spring Cloud Vault 将使用可用的配置文件名称作为附加到指定基本路径的后缀，搜索其中的键/值对。

它还将在可配置的默认应用程序路径下查找属性（使用或不使用配置文件后缀），这样我们就可以在一个位置拥有共享机密。请谨慎使用此功能！

总而言之，如果fakebank 应用程序的生产配置文件处于活动状态，Spring Vault将查找存储在以下路径下的属性：
    1. secret/fakebank/production（更高优先级）
    2. secret/fakebank
    3. secret/application/production
    4. secret/application（优先级较低）

在前面的列表中，application是Spring用作Secret的默认附加位置的名称。我们可以使用spring.cloud.vault.generic.default-context属性对其进行修改。

## 5.使用数据库密钥后端
数据库后端模块允许Spring应用程序使用由Vault创建的动态生成的数据库凭据。Spring Vault在标准的spring.datasource.username和spring.datasource.password属性下注入这些凭据，因此可以通过常规DataSource选择它们。

请注意，在使用此后端之前，我们必须在Vault中创建数据库配置和角色，如[上一个教程](https://blog.csdn.net/peterwanghao/article/details/83181932)中所述。

为了在Spring应用程序中使用Vault生成的数据库凭据，spring-cloud-vault-config-databases必须存在于项目的类路径中，以及相应的JDBC驱动程序。

我们还需要通过在bootstrap.yml中添加一些属性来在我们的应用程序中使用它：
```
spring:
  cloud:
    vault:
      database:
        enabled: true
        role: fakebank-accounts-ro
        backend: database
```
这里最重要的属性是role属性，它包含存储在Vault中的数据库角色名称。在引导期间，Spring将联系Vault并请求它创建具有相应权限的新凭据。

默认情况下，保管库将在配置的生存时间之后撤消与这些凭据关联的权限。

幸运的是，Spring Vault将自动续订与获取的凭据相关的租约。通过这样做，只要我们的应用程序运行，凭据就会保持有效。

现在，让我们看看这种整合的实际效果。以下代码段从Spring管理的DataSource获取新的数据库连接：
```
Connection c = datasource.getConnection()
```


再一次，我们可以看到代码中没有使用Vault的迹象。所有集成都发生在环境级别，因此我们的代码可以像往常一样轻松进行单元测试。

## 6.结论
在本教程中，我们展示了如何使用Spring Vault库将Vault与Spring Boot集成。我们已经介绍了两个常见用例：通用键/值对和动态数据库凭据。

