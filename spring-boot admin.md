# spring-boot admin

## 服务器端

### 版本

spring boot admin服务器必须严格按照文档的有要求来搭配spring boot，否则可能出现各种问题，例如：spring boot admin 2.2.3这个版本，按照文档是应该搭配spring boot 2.2.7和[Hoxton.SR4，如果你搭配spring boot 2.2.8就会发生tcp的大量closed状态连接，而且连接会占用大量的资源，直到耗尽服务器。服务器端的版本必须严格验证文档限制，而客户端要没有特别严格的版本现在，但应保证在大版本范围内，例如：客户端spring boot 2.2.x和Hoxton.SRX。

### pom.xml

```xml
	<!-- spring boot parent -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.2.7.RELEASE</version>
	</parent>

	<!--spring cloud dependencis -->
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Hoxton.SR4</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<!-- 相关依赖包 -->
	<dependencies>
		<!-- spring cloud config client -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<!-- spring cloud bus -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
		</dependency>	
		<!-- spring boot security -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>	
		 
		<!-- spring boot actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- spring boot web -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>		
		<!-- spring boot admin -->
		<dependency>
		    <groupId>de.codecentric</groupId>
		    <artifactId>spring-boot-admin-starter-server</artifactId>
		    <version>2.2.3</version>
		</dependency>
        
		<!-- spring cloud eureka 为了从eureka中获取客户端actuator信息 -->
		<dependency>
		    <groupId>org.springframework.cloud</groupId>
		    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>        
					
	</dependencies>
```

spring boot admin可以有两种方式来监控客户端actuator方式，第一种是客户端和服务器端直接通讯方式，客户端要知道spring boot admin的url地址，把自己注册到spring boot admin服务器。第二种就是服务器端和客户端都通过eureka来注册获取对方的actuator位置信息，建议使用第二种。

### yaml

```yaml
spring:
  # 开启安全认证       
  security:
    user:
      name: dy-admin
      password: 12345678
      
# eureka 客户端
eureka:
  instance: 
    # 使用ip地址注册到eureka服务器(多ip的情况下和bootstrap.xml的spring.inetutils.preferred-networks属性配合使用),默认值false使用主机名注册(/etc/hosts的第一行)
    prefer-ip-address: true
    # 注册到eureka服务器的实例id,格式为:本机ip地址:服务名:端口(多ip情况下和bootstrap.xml的spring.inetutils.preferred-networks属性配合使用)
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
    # 注册security用户名和密码,供spring boot admin访问
    metadata-map:
      # 当前应用配置的spring security用户名和密码
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}  
  client:
    service-url:
      # eureka注册中心位置
      defaultZone: http://dy-eureka:12345678@192.168.5.76:8761/eureka/        
```

spring.security.user.name和spring.security.user.password就是spring boot admin的登录账户和密码。

服务器的eureka的作用有两个：

1.把自己也注册到eureka上，spring boot admin也可以监控自己。

2.从eureka上获取(发现)所有使用spring boot admin client的应用信息，用于监控。



### @EnableAdminServer

```java
@SpringBootApplication
@EnableAdminServer
public class BootAdminServerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(BootAdminServerApplication.class, args);
	}

}

```

### WebSecurityConfig

下面的一个成熟的配置，如果你已经开启了spring security，则下面的WebSecurityConfig和代码是必须的。

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

	private final String adminContextPath;

	public WebSecurityConfig(AdminServerProperties adminServerProperties) {
		this.adminContextPath = adminServerProperties.getContextPath();
	}

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		// @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter( "redirectTo" );

        http.authorizeRequests()
                .antMatchers( adminContextPath + "/assets/**" ).permitAll()
                .antMatchers( adminContextPath + "/login" ).permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage( adminContextPath + "/login" ).successHandler( successHandler ).and()
                .logout().logoutUrl( adminContextPath + "/logout" ).and()
                .httpBasic().and()
                .csrf().disable();
        // @formatter:on
	}
}
```



## 客户端

### pom.xml

```xml
		<!-- spring boot admin client -->
		<dependency>
		    <groupId>de.codecentric</groupId>
		    <artifactId>spring-boot-admin-starter-client</artifactId>
		    <version>2.2.3</version>
		</dependency>    

		<!-- spring cloud eureka client -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>

		<!-- spring boot actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
```

建议客户端和服务器同一个版本号。



### yaml

#### 直连spring boot admin(非eureka环境)

直连试用于没有eureka的环境，例如：spring cloud config。

```yaml
spring:
  # spring boot admin client
  boot:
    admin:
      client:
        # spring boot admin服务器地址URL，访问用户名和密码
        url: http://192.168.5.76:8000
        username: dy-admin
        password: 12345678          
        instance:
          metadata:
			# 当前应用配置的spring security用户名和密码
            user.name: ${spring.security.user.name}
            user.password: ${spring.security.user.password}    
          prefer-ip: true
          # 当前应用的业务url(主页)
          service-url: http://${spring.cloud.client.ip-address}:${server.port}
          # 当前应用的/actuator
          management-url:  http://${spring.cloud.client.ip-address}:${server.port}/actuator

```

#### 通过eureka连接到spring boot admin(建议)

eureka环境比较简单，只需要在eureka的instance上加入metadata-map.user就可以了。

建议：即使不用eureka客户端的管理系统为了能让spring boot admin监控上最好也使用eureka客户端注册到eureka上，方便spring boot admin监控。

```yaml
eureka:
  instance: 
	# 注册security用户名和密码,供spring boot admin访问
    metadata-map:
      # 当前应用配置的spring security用户名和密码
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}  
```

使用eureka配置就比使用直连的配置简单的多，因为spring boot admin server和client在启动的时候把自己相关的属性都注册到eureka上，并发现(eureka发现)对方的注册信息(spring boot admin相关属性)，然后为自己所用。

服务器端：服务器端启动的时候，会注册访问url、security.user和security.password到eureka上，并发现所用注册客户端的地址、security.user.name、security.user.password、actuator地址等。

客户端：客户端在启动的时候，会注册自己的security.user.name、security.user.password(通过eureka.instance.metadata-map)、注册actuator的访问url，并发现服务器端访问url、security.user.name和security.user.password。好文章

#### Spring Security开启/actuator保护

```yaml
spring:
  # 开启安全认证       
  security:
    user:
      name: dfss-fss
      password: 12345678
```



```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity httpSecurity) throws Exception {
		// actuator访问需要basic认证,并且禁用CSRF防御
		// @formatter:off
		httpSecurity.authorizeRequests()
		.antMatchers("/actuator/**").authenticated()
		.antMatchers("/**").permitAll()
		.and().httpBasic()
		.and().csrf().disable();
		// @formatter:on
	}
}
```

```
/actuator/**规则写在最前前面其会被优先执行，这里定义为需要认证；
/**规则写在/actuator/**规则的后面，其会在/actuator/**规则的后面执行，这里定义为不需要任何授权就可以访问；
.and().httpBasic()，定义了认证模式http basic模式；
.and().csrf().disable(); 定义了禁止csrf；
```





## FAQ

### 监控应用变红

当前监控的应用变红说明这个应用或服务以及掉线了，或者调用这个应用的健康检查(/actuator/health)返回了非UP状态。

### 通过查看日志得到应有状态

你可以通过查看服务器的**日志报表**来查看某个应用和服务当前的状态，例如：DOWN，为什么是红色标志等。

其时通过发送/actuator/health请求，并根据判断状态返回值StatusInfo.status=UP来判断应用是否健康的，健康为绿色，不健康为红色。

例如：掉线日志

```json
{
    "statusInfo": {
        "status": "OFFLINE",
        "details": {
            "exception": "io.netty.channel.AbstractChannel$AnnotatedConnectException",
            "message": "finishConnect(..) failed: 拒绝连接: /192.168.5.76:5500"
        }
    }
}
```

例如：上线日志

```json
{
    "statusInfo": {
        "status": "UP"
    }    
}
```

### CPU消耗大

注意，如果你使用spring boot admin实时监控某个服务和应用，则cpu消耗百分比马上就上去。这个要注意，如果没必要不要在生产环境下监控，例如：查看jvm运行情况，cpu的使用率会从0.3%上升到56%。

