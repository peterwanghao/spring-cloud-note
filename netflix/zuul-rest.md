# 使用Spring Cloud Netflix Zuul代理网关访问后台REST服务

## 1.概述

在本文中，我们将探讨如何在互相单独部署的前端应用程序和后端REST API服务之间进行通信。目的是解决浏览器的跨域资源访问和同源策略限制，允许页面UI能够调用后台的API，即使它们不在同一个服务器中。

我们在这里创建了两个独立的应用程序 - 一个UI应用程序和一个简单的REST API，我们将在UI应用程序中使用Zuul代理来代理对REST API的调用。Zuul是Netflix基于JVM的路由器和服务器端负载均衡器。Spring Cloud与嵌入式Zuul代理有很好的集成。

## 2.REST应用

我们的REST API应用程序是一个简单的Spring Boot应用程序。在本文中将在端口8081上运行服务器中部署的API 。

配置文件

```
server.contextPath=/spring-zuul-foos-resource
server.port=8081
```

让我们首先为我们将要使用的资源定义基本DTO：

```
public class Foo {
    private long id;
    private String name;
 
    // standard getters and setters
}
```

定义一个简单的控制器：

```
@Controller
public class FooController {
 
    @RequestMapping(method = RequestMethod.GET, value = "/foos/{id}")
    @ResponseBody
    public Foo findById(
      @PathVariable long id, HttpServletRequest req, HttpServletResponse res) {
        return new Foo(Long.parseLong(randomNumeric(2)), randomAlphabetic(4));
    }
}
```

## 3.前端应用程序

我们的UI应用程序也是一个简单的Spring Boot应用程序。在本文中此应用运行在端口8080上。

首先，我们需要通过Spring Cloud向我们的UI应用程序的pom.xml添加对zuul支持的依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

接下来 - 我们需要配置Zuul，因为我们正在使用Spring Boot，所以我们将在application.yml中执行此操作：

```
zuul:
  routes:
    foos:
      path: /foos/**
      url: http://localhost:8081/spring-zuul-foos-resource/foos
```

注意：

 - 我们代理我们的资源服务器Foos。 
 - 来自UI的所有以“ / foos / ”开头的请求将被路由到我们的Foos资源服务器，地址为：http：// loclahost：8081/spring-zuul-foos-resource / foos /

然后编写我们的主页面index.html — 这里使用一些AngularJS：

```
<html>
<body ng-app="myApp" ng-controller="mainCtrl">
<script src="angular.min.js"></script>
<script src="angular-resource.min.js"></script>
 
<script>
var app = angular.module('myApp', ["ngResource"]);
 
app.controller('mainCtrl', function($scope,$resource,$http) {
    $scope.foo = {id:0 , name:"sample foo"};
    $scope.foos = $resource("/foos/:fooId",{fooId:'@id'});
     
    $scope.getFoo = function(){
        $scope.foo = $scope.foos.get({fooId:$scope.foo.id});
    }  
});
</script>
 
<div>
    <h1>Foo Details</h1>
    <span>{{foo.id}}</span>
    <span>{{foo.name}}</span>
    <a href="#" ng-click="getFoo()">New Foo</a>
</div>
</body>
</html>
```

这里最重要的方面是我们如何使用相对URL访问API ！
请记住，API应用程序未部署在与UI应用程序相同的服务器上，因此相对URL不起作用，并且在没有代理的情况下不起作用。
但是，通过代理，我们通过Zuul代理访问Foo资源，Zuul代理配置为将这些请求路由到实际部署API的位置。

最后，启用Boot的应用程序：

```
@EnableZuulProxy
@SpringBootApplication
public class UiApplication extends SpringBootServletInitializer {
 
    public static void main(String[] args) {
        SpringApplication.run(UiApplication.class, args);
    }
}
```

这里使用@EnableZuulProxy注解来启动Zuul代理，这非常干净和简洁。

## 4.运行测试

分别启动2个应用系统，在浏览器中输入http://localhost:8080/index 
每点击一次“New Foo”按钮就访问后台REST API一次。
![这里写图片描述](https://img-blog.csdn.net/2018091113445319?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BldGVyd2FuZ2hhbw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 5.定制Zuul过滤器

有多个Zuul过滤器可用，我们也可以创建自己的定制过滤器：

```
@Component
public class CustomZuulFilter extends ZuulFilter {
 
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.addZuulRequestHeader("Test", "TestSample");
        return null;
    }
 
    @Override
    public boolean shouldFilter() {
       return true;
    }
    // ...
}
```

这个简单的过滤器只是在请求头中添加了一个名为“ Test ” 的属性- 当然，我们可以根据需要增加我们的请求。

## 6.测试自定义Zuul过滤器

最后，让我们测试一下，确保我们的自定义过滤器正常工作 - 首先我们将在Foos资源服务器上修改我们的FooController：

```
@Controller
public class FooController {
 
    @GetMapping("/foos/{id}")
    @ResponseBody
    public Foo findById(
      @PathVariable long id, HttpServletRequest req, HttpServletResponse res) {
        if (req.getHeader("Test") != null) {
            res.addHeader("Test", req.getHeader("Test"));
        }
        return new Foo(Long.parseLong(randomNumeric(2)), randomAlphabetic(4));
    }
}
```

现在 - 让我们测试一下：

```
@Test
public void whenSendRequest_thenHeaderAdded() {
    Response response = RestAssured.get("http://localhost:8080/foos/1");
  
    assertEquals(200, response.getStatusCode());
    assertEquals("TestSample", response.getHeader("Test"));
}
```

## 7.结论

在这篇文章中，我们专注于使用Zuul将请求从UI应用程序路由到REST API。我们成功地解决了CORS和同源策略，我们还设法定制和扩充了传输中的HTTP请求。