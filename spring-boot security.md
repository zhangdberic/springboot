# spring boot security

## pom.xml

```xml
		<!-- spring boot security -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
```

## 配置用户名和密码登录认证

yaml配置文件，加入如下内容，配置登录安全认证用户名和密码，即使没有下面的配置只要加入了spring-boot-starter-security依赖包，安全认证就已经启动了：

```yaml
spring:
  # 开启安全认证       
  security:
    user:
      name: dy-eureka
      password: 12345678
```

## 基于http basic方式登录认证

默认基于http普通页面的登录认证，其有一个最大的缺点就是登录html页面引用了国外网址的css，造成访问登录非常慢，这里我们修改登录认证模式为http basic。

```java
    @EnableWebSecurity
    class WebSecurityConfig extends WebSecurityConfigurerAdapter {
        @Override
        protected void configure(HttpSecurity httpSecurity) throws Exception {
		httpSecurity.authorizeRequests().anyRequest().authenticated().and().httpBasic();
        }
    }
```

## spring boot 2.0无效的配置属性

```
security.basic.authorize-mode
security.basic.enabled
security.basic.path
security.basic.realm
security.enable-csrf
security.headers.cache
security.headers.content-security-policy
security.headers.content-security-policy-mode
security.headers.content-type
security.headers.frame
security.headers.hsts
security.headers.xss
security.ignored
security.require-ssl
security.sessions
```

