# 第一节 微服务的安全保护

在Spring Cloud中对应用进行安全保护通常使用Spring Security，这种方式集成起来非常简单而且很容易扩展现有的应用场景。在分布式环境中Spring Security使用Spring Session和Redis来共享会话。共享会话可以将在微服务网关中登录的用户验证信息传递到系统中的其它服务中，在微服务中实现单点认证。

## 1. 配置
首先在每个模块工程中都增加spring-boot-starter-security的引用。
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

然后在discovery，gateway，book-service和rating-service四个工程中增加spring-session和Redis的引用。
```
<dependency>
	<groupId>org.springframework.session</groupId>
	<artifactId>spring-session</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

接下来，在这四个工程中新建一个会话配置类，此类同主应用类文件放在同一个目录下。
```
@Configuration
@EnableRedisHttpSession
public class SessionConfig extends AbstractHttpSessionApplicationInitializer {
}
```

最后在Git仓库中保存的各个*.properties文件里增加redis配置。
```
spring.redis.host=localhost
spring.redis.port=6379
```

## 2. 配置中心的安全保护
配置中心通常保存很多敏感数据，比如数据库的连接配置、第三方API接口的KEY等。因此需要对配置中心的访问进行秘密保护。修改src/main/resources目录下的application.properties文件。
```
eureka.client.serviceUrl.defaultZone=http://discUser:discPassword@localhost:8082/eureka/

security.user.name=configUser
security.user.password=configPassword
security.user.role=SYSTEM
```
这样就可以通过用户名、密码来进行访问控制，同时注册到服务中心时也需要认证。

## 3. 服务注册中心的安全保护
服务注册中心会保存系统中各个服务的地址信息，如果恶意的客户端获得了访问权，它们将访问到我们系统中所有服务的网络位置，并能够将他们自己的恶意服务注册到我们的应用程序中。因此确保服务注册中心的安全是至关重要的。

### 3.1 服务端点的安全保护
让我们添加一个安全过滤器来保护其他服务将使用的端点。这个过滤器为服务注册中心配置了一个隔离的身份认证环境。
```
@Configuration
@EnableWebSecurity
@Order(1)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
 
   @Autowired
   public void configureGlobal(AuthenticationManagerBuilder auth) {
       auth.inMemoryAuthentication().withUser("discUser")
         .password("discPassword").roles("SYSTEM");
   }
 
   @Override
   protected void configure(HttpSecurity http) {
       http.sessionManagement()
         .sessionCreationPolicy(SessionCreationPolicy.ALWAYS)
         .and().requestMatchers().antMatchers("/eureka/**")
         .and().authorizeRequests().antMatchers("/eureka/**")
         .hasRole("SYSTEM").anyRequest().denyAll().and()
         .httpBasic().and().csrf().disable();
   }
}
```
在这里为我们的服务创建了一个角色为‘SYSTEM’的用户，访问注册中心的用户必须拥有这个角色，同时还增加了一些安全控制：

- @Order(1) — 告诉Spring这个安全过滤器第一个执行
- sessionCreationPolicy — 当一个用户登录时总是创建一个会话
- requestMatchers — 限制该过滤器应用的端点

### 3.2 Eureka仪表盘的安全保护
Eureka提供一个UI界面的仪表盘来查看当前注册的服务，因此也需要对它进行安全保护。这里我们使用第二个安全过滤器，需要注意的是这个过滤器没有@Order()标签代表是最后执行的安全过滤器。
```
@Configuration
public static class AdminSecurityConfig
  extends WebSecurityConfigurerAdapter {
 
@Override
protected void configure(HttpSecurity http) {
   http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER)
     .and().httpBasic().disable().authorizeRequests()
     .antMatchers(HttpMethod.GET, "/").hasRole("ADMIN")
     .antMatchers("/info", "/health").authenticated().anyRequest()
     .denyAll().and().csrf().disable();
   }
}
```
这个过滤器有一些特定的设置：

- httpBasic().disable() — 告诉Spring Security禁用该过滤器的所有身份验证过程
- sessionCreationPolicy — 我们将其设置为NEVER表明用户需要在访问由该过滤器保护的资源之前已经经过身份验证。

### 3.3 配置中心认证
在src/main/resources目录下的bootstrap.properties文件里增加2条配置，代表访问配置信息时需要进行身份认证。
```
spring.cloud.config.username=configUser
spring.cloud.config.password=configPassword
```

修改Git仓库中的discovery.properties文件
```
eureka.client.serviceUrl.defaultZone=http://discUser:discPassword@localhost:8082/eureka/
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
```

## 4. 服务网关的安全保护
服务网关是我们系统面向外部的窗口，因此希望只有经过身份认证的用户才能访问后面的敏感信息。

### 4.1 安全配置
新建一个SecurityConfig类，添加如下内容：
```
@EnableWebSecurity
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

	@Autowired
	public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
		auth.inMemoryAuthentication().withUser("peter").password("123456").roles("USER").and().withUser("admin")
				.password("admin").roles("ADMIN");
	}

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.formLogin().defaultSuccessUrl("/home/index.html", true).and().authorizeRequests()
				.antMatchers("/book-service/**", "/rating-service/**", "/login*", "/").permitAll()
				.antMatchers("/eureka/**").hasRole("ADMIN").anyRequest().authenticated().and().logout().and().csrf()
				.disable();
	}
}
```
我们首先建立了2个用户分配不同的角色，然后创建一个安全过滤器，使用Form表单登录来确保各个端点的安全。

然后在配置类里增加@EnableRedisHttpSession注解
```
@EnableRedisHttpSession(redisFlushMode = RedisFlushMode.IMMEDIATE)
```
将会话刷新模式设为立即更新，这样有助于重定向时准备认证令牌。

最后增加一个ZuulFilter处理登录后的认证令牌。该过滤器将在登录后重定向请求并将session保存到Cookie中，这将帮助认证信息传递到后面的服务中心。
```
@Component
public class SessionSavingZuulPreFilter extends ZuulFilter {

	private Logger log = LoggerFactory.getLogger(this.getClass());

	@Autowired
	private SessionRepository repository;

	@Override
	public boolean shouldFilter() {
		return true;
	}

	@Override
	public Object run() {
		RequestContext context = RequestContext.getCurrentContext();
		HttpSession httpSession = context.getRequest().getSession();
		Session session = repository.getSession(httpSession.getId());

		context.addZuulRequestHeader("Cookie", "SESSION=" + httpSession.getId());
		log.info("ZuulPreFilter session proxy: {}", session.getId());
		return null;
	}

	@Override
	public String filterType() {
		return "pre";
	}

	@Override
	public int filterOrder() {
		return 0;
	}
}
```
### 4.2 配置中心和服务中心认证
在网关服务的src/main/resources目录下修改bootstrap.properties文件
```
spring.cloud.config.username=configUser
spring.cloud.config.password=configPassword
 eureka.client.serviceUrl.defaultZone=http://discUser:discPassword@localhost:8082/eureka/
```

修改Git仓库中的gateway.properties文件
```
management.security.sessions=always
 
zuul.routes.book-service.path=/book-service/**
zuul.routes.book-service.sensitive-headers=Set-Cookie,Authorization
hystrix.command.book-service.execution.isolation.thread
    .timeoutInMilliseconds=600000
 
zuul.routes.rating-service.path=/rating-service/**
zuul.routes.rating-service.sensitive-headers=Set-Cookie,Authorization
hystrix.command.rating-service.execution.isolation.thread
    .timeoutInMilliseconds=600000
 
zuul.routes.discovery.path=/discovery/**
zuul.routes.discovery.sensitive-headers=Set-Cookie,Authorization
zuul.routes.discovery.url=http://localhost:8082
hystrix.command.discovery.execution.isolation.thread
    .timeoutInMilliseconds=600000
```
## 5. 图书服务安全保护
### 5.1 安全配置
创建一个安全配置类
```
@EnableWebSecurity
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

	@Autowired
	public void configureGlobal1(AuthenticationManagerBuilder auth) throws Exception {
		// try in memory auth with no users to support the case that this will allow for
		// users that are logged in to go anywhere
		auth.inMemoryAuthentication();
	}

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.httpBasic().disable().authorizeRequests()
	      .antMatchers("/books").permitAll()
	      .antMatchers("/books/*").hasAnyRole("USER", "ADMIN").anyRequest()
	      .authenticated().and().csrf().disable();
	}
}
```

### 5.2 参数
在图书服务的src/main/resources目录下修改bootstrap.properties文件
```
spring.cloud.config.username=configUser
spring.cloud.config.password=configPassword
 eureka.client.serviceUrl.defaultZone=http://discUser:discPassword@localhost:8082/eureka/
```

修改Git仓库中的book-service.properties文件
```
management.security.sessions=never
```

## 6. 评分服务安全保护
评分服务的安全保护类似于图书服务，也是新建一个安全配置类并增加一些参数。

## 7. 测试
启动Redis和所有的服务：config, discovery, gateway, book-service, and rating-service. 

在gateway工程里编写测试用例来验证。
```
/**
	 * 
	 * @Title: whenGetAllBooks_thenSuccess
	 * @Description: 访问一个没有权限控制的接口
	 * @param: 
	 * @return: void
	 * @throws
	 */
	@Test
	public void whenGetAllBooks_thenSuccess() {
		TestRestTemplate testRestTemplate = new TestRestTemplate();

		ResponseEntity<String> response = testRestTemplate.getForEntity(ROOT_URI + "/book-service/books", String.class);
		Assert.assertEquals(HttpStatus.OK, response.getStatusCode());
		Assert.assertNotNull(response.getBody());
	}

	/**
	 * 
	 * @Title: whenAccessProtectedResourceWithoutLogin_thenForbidden
	 * @Description: 访问一个有权限控制的接口，返回403
	 * @param: 
	 * @return: void
	 * @throws
	 */
	@Test
	public void whenAccessProtectedResourceWithoutLogin_thenForbidden() {
		TestRestTemplate testRestTemplate = new TestRestTemplate();
		ResponseEntity<String> response = testRestTemplate.getForEntity(ROOT_URI + "/book-service/books/1",
				String.class);
		Assert.assertEquals(HttpStatus.FORBIDDEN, response.getStatusCode());
	}

	/**
	 * 
	 * @Title: whenAccessProtectedResourceBySession_thenSuccess
	 * @Description: 访问一个有权限控制的接口，通过共享会话可正常访问受保护资源
	 * @param: 
	 * @return: void
	 * @throws
	 */
	@Test
	public void whenAccessProtectedResourceBySession_thenSuccess() {
		TestRestTemplate testRestTemplate = new TestRestTemplate();
		MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
		form.add("username", "peter");
		form.add("password", "123456");
		ResponseEntity<String> response = testRestTemplate.postForEntity(ROOT_URI + "/login", form, String.class);

		String sessionCookie = response.getHeaders().get("Set-Cookie").get(0).split(";")[0];
		HttpHeaders headers = new HttpHeaders();
		headers = new HttpHeaders();
		headers.add("Cookie", sessionCookie);
		HttpEntity<String> httpEntity = new HttpEntity<>(headers);

		response = testRestTemplate.exchange(ROOT_URI + "/book-service/books/1", HttpMethod.GET, httpEntity,
				String.class);
		Assert.assertEquals(HttpStatus.OK, response.getStatusCode());
		Assert.assertNotNull(response.getBody());
	}

	/**
	 * 
	 * @Title: whenAccessProtectedResourceAfterLogin_thenSuccess
	 * @Description: 登录后访问受保护资源，可正常访问
	 * @param: 
	 * @return: void
	 * @throws
	 */
	@Test
	public void whenAccessProtectedResourceAfterLogin_thenSuccess() {
		final Response response = RestAssured.given().auth().form("peter", "123456", formConfig)
				.get(ROOT_URI + "/book-service/books/1");
		Assert.assertEquals(HttpStatus.OK.value(), response.getStatusCode());
		Assert.assertNotNull(response.getBody());
		System.out.println(response.getBody().asString());
	}

	/**
	 * 
	 * @Title: whenAccessAdminProtectedResource_thenForbidden
	 * @Description: 使用普通用户权限访问管理员资源，拒绝访问
	 * @param: 
	 * @return: void
	 * @throws
	 */
	@Test
	public void whenAccessAdminProtectedResource_thenForbidden() {
		final Response response = RestAssured.given().auth().form("peter", "123456", formConfig)
				.get(ROOT_URI + "/rating-service/ratings");
		Assert.assertEquals(HttpStatus.FORBIDDEN.value(), response.getStatusCode());

	}
```

## 8. 总结
云计算的安全性非常复杂，但在Spring Security和Spring Session的帮助下，我们可以很容易地解决这个问题。现在我们有了一个具有安全保护的云服务，使用Zuul和Spring Session可以只在一个服务中进行身份认证，然后将认证信息传播到整个应用中。这意味着我们可以很容易地将应用程序拆分成适当的域，并以我们认为合适的方式对它们进行保护。