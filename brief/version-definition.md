# 第三节 版本定义

Spring Cloud是一个由众多独立子项目组成的大型综合项目，每个子项目有不同的发行节奏，都维护着自己的发布版本号。Spring Cloud通过一个资源清单BOM（Bill of Materials）来管理每个版本的子项目清单。为避免与子项目的发布号混淆，所以没有采用版本号的方式，而是通过命名的方式。

这些版本名称的命名方式采用了伦敦地铁站的名称，同时根据字母表的顺序来对应版本时间顺序，比如：最早的Release版本：Angel，第二个Release版本：Brixton，然后是Camden、Dalston、Edgware，目前最新的是Finchley版本。

当一个版本的Spring Cloud项目的发布内容积累到临界点或者解决了一个严重bug后，就会发布一个“service releases”版本，简称SRX版本，其中X是一个递增数字。当前官网上最新的稳定版本是Edgware.SR2，里程碑版本是Finchley.M7。下表列出了这两个版本所包含的子项目及各子项目的版本号。

| Component                 | Edgware.SR2    | Finchley.M7  | Finchley.BUILD-SNAPSHOT |
| ------------------------- | -------------- | ------------ | ----------------------- |
| spring-cloud-aws          | 1.2.2.RELEASE  | 2.0.0.M4     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-bus          | 1.3.2.RELEASE  | 2.0.0.M6     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-cli          | 1.4.1.RELEASE  | 2.0.0.M1     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-commons      | 1.3.2.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-contract     | 1.2.3.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-config       | 1.4.2.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-netflix      | 1.4.3.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-security     | 1.2.2.RELEASE  | 2.0.0.M2     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-cloudfoundry | 1.1.1.RELEASE  | 2.0.0.M3     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-consul       | 1.3.2.RELEASE  | 2.0.0.M6     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-sleuth       | 1.3.2.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-stream       | Ditmars.SR3    | Elmhurst.RC1 | Elmhurst.BUILD-SNAPSHOT |
| spring-cloud-zookeeper    | 1.2.0.RELEASE  | 2.0.0.M6     | 2.0.0.BUILD-SNAPSHOT    |
| spring-boot               | 1.5.10.RELEASE | 2.0.0.RC2    | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-task         | 1.2.2.RELEASE  | 2.0.0.M3     | 2.0.0.RELEASE           |
| spring-cloud-vault        | 1.1.0.RELEASE  | 2.0.0.M6     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-gateway      | 1.0.1.RELEASE  | 2.0.0.M7     | 2.0.0.BUILD-SNAPSHOT    |
| spring-cloud-openfeign    |                | 2.0.0.M1     | 2.0.0.BUILD-SNAPSHOT    |

Finchley 与 Spring Boot 2.0.x, 兼容，不支持 Spring Boot 1.5.x.
Dalston 和 Edgware 与 Spring Boot 1.5.x, 兼容，不支持 Spring Boot 2.0.x.
Camden 是构建在 Spring Boot 1.4.x, 之上，但也支持 1.5.x.
Brixton 是构建在 Spring Boot 1.3.x, 之上，但也支持 1.4.x.
Angel 是构建在 Spring Boot 1.2.x, 之上，但也兼容 Spring Boot 1.3.x.
**注意: Angel 和 Brixton 两个版本已于2017年7月终止不再进行维护。**