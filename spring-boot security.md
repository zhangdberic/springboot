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

## POST提交CSRF

如果你加载了spring-boot-starter-security那么安全验证就默认已经开启了，默认csrf是true，也就是说所有的POST提交都要有csrf_token，但很多情况下提交是没有csrf的那么就需要禁用，例如：/actuator/bus-refresh的POST提交.

```
		httpSecurity.csrf().disable();
```

## httpBasic()

```
httpSecurity.httpBasic();
```

httpBasic()这个方法相当于加入了一个http basic验证的过滤器(BasicAuthenticationFilter)，其优先级级在URL匹配允许和禁止访问过滤器前执行。

注意：请求url如果被deny()定义的url规则匹配到，则不会马上返回403而是先要求用户登录(basic)，然后即使登录成功也会抛出403。经过调试代码发现，因为deny()匹配到的url，一定会触发access deny异常，而这个抛出后，BasicAuthenticationEntryPoint会被执行，因此就有了即使已经被deny()定义的url匹配上，但还是要求你先basic登录的问题。

## 核心代码

### HttpSecurity

你对httpSecurity的方法调用，就是执行加入、删除和配置安全过滤器的过程，例如：csrf().disable()删除了CsrfFilter过滤器，httpBasic()加入了BasicAuthenticationFilter过滤器。

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity httpSecurity) throws Exception {
		// actuator访问需要basic认证,并且禁用CSRF防御
		//@formatter:off
		httpSecurity.authorizeRequests()
		.antMatchers("/actuator/**").authenticated()
		.antMatchers("/services/**").permitAll()
		.antMatchers("/**").denyAll()
		.and().httpBasic()
		.and().csrf().disable();
		 //@formatter:on
	}
}
```



### 过滤器执行顺序(FilterComparator)

每个过滤器的排序号(ORDER)相差为100。

```java
	FilterComparator() {
		Step order = new Step(INITIAL_ORDER, ORDER_STEP);
		put(ChannelProcessingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(WebAsyncManagerIntegrationFilter.class, order.next());
		put(SecurityContextPersistenceFilter.class, order.next());
		put(HeaderWriterFilter.class, order.next());
		put(CorsFilter.class, order.next());
		put(CsrfFilter.class, order.next());
		put(LogoutFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationRequestFilter",
				order.next());
		put(X509AuthenticationFilter.class, order.next());
		put(AbstractPreAuthenticatedProcessingFilter.class, order.next());
		filterToOrder.put("org.springframework.security.cas.web.CasAuthenticationFilter",
				order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationFilter",
				order.next());
		put(UsernamePasswordAuthenticationFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.openid.OpenIDAuthenticationFilter", order.next());
		put(DefaultLoginPageGeneratingFilter.class, order.next());
		put(DefaultLogoutPageGeneratingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(DigestAuthenticationFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationFilter", order.next());
		put(BasicAuthenticationFilter.class, order.next());
		put(RequestCacheAwareFilter.class, order.next());
		put(SecurityContextHolderAwareRequestFilter.class, order.next());
		put(JaasApiIntegrationFilter.class, order.next());
		put(RememberMeAuthenticationFilter.class, order.next());
		put(AnonymousAuthenticationFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationCodeGrantFilter",
				order.next());
		put(SessionManagementFilter.class, order.next());
		put(ExceptionTranslationFilter.class, order.next());
		put(FilterSecurityInterceptor.class, order.next());
		put(SwitchUserFilter.class, order.next());
	}
```

通过开启调试org.springframework.security的日志配置，我们可以看到一个请求经过的过滤器顺序，例如：

/services请求，这个配置了permitAll()，如下是我为了查看方便抽取的过滤器执行顺序，eclipse打印的日志在下面：

```
WebAsyncManagerIntegrationFilter
SecurityContextPersistenceFilter
HeaderWriterFilter
LogoutFilter
BasicAuthenticationFilter
RequestCacheAwareFilter
SecurityContextHolderAwareRequestFilter
AnonymousAuthenticationFilter
SessionManagementFilter
ExceptionTranslationFilter
FilterSecurityInterceptor
```

eclipse打印的日志：

```
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 1 of 11 in additional filter chain; firing Filter: 'WebAsyncManagerIntegrationFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 2 of 11 in additional filter chain; firing Filter: 'SecurityContextPersistenceFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.context.HttpSessionSecurityContextRepository - Obtained a valid SecurityContext from SPRING_SECURITY_CONTEXT: 'org.springframework.security.core.context.SecurityContextImpl@4e9c2f63: Authentication: org.springframework.security.authentication.UsernamePasswordAuthenticationToken@4e9c2f63: Principal: org.springframework.security.core.userdetails.User@b163eb6b: Username: dy-sgw; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Not granted any authorities; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@3bcc: RemoteIpAddress: 192.168.5.31; SessionId: null; Not granted any authorities'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 3 of 11 in additional filter chain; firing Filter: 'HeaderWriterFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 4 of 11 in additional filter chain; firing Filter: 'LogoutFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.OrRequestMatcher - Trying to match using Ant [pattern='/logout', GET]
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Checking match of request : '/services'; against '/logout'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.OrRequestMatcher - Trying to match using Ant [pattern='/logout', POST]
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Request 'GET /services' doesn't match 'POST /logout'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.OrRequestMatcher - Trying to match using Ant [pattern='/logout', PUT]
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Request 'GET /services' doesn't match 'PUT /logout'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.OrRequestMatcher - Trying to match using Ant [pattern='/logout', DELETE]
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Request 'GET /services' doesn't match 'DELETE /logout'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.OrRequestMatcher - No matches found
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 5 of 11 in additional filter chain; firing Filter: 'BasicAuthenticationFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.authentication.www.BasicAuthenticationFilter - Basic Authentication Authorization header found for user 'dy-sgw'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 6 of 11 in additional filter chain; firing Filter: 'RequestCacheAwareFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.savedrequest.HttpSessionRequestCache - saved request doesn't match
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 7 of 11 in additional filter chain; firing Filter: 'SecurityContextHolderAwareRequestFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 8 of 11 in additional filter chain; firing Filter: 'AnonymousAuthenticationFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.authentication.AnonymousAuthenticationFilter - SecurityContextHolder not populated with anonymous token, as it already contained: 'org.springframework.security.authentication.UsernamePasswordAuthenticationToken@4e9c2f63: Principal: org.springframework.security.core.userdetails.User@b163eb6b: Username: dy-sgw; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Not granted any authorities; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@3bcc: RemoteIpAddress: 192.168.5.31; SessionId: null; Not granted any authorities'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 9 of 11 in additional filter chain; firing Filter: 'SessionManagementFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 10 of 11 in additional filter chain; firing Filter: 'ExceptionTranslationFilter'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services at position 11 of 11 in additional filter chain; firing Filter: 'FilterSecurityInterceptor'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Checking match of request : '/services'; against '/actuator/**'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.util.matcher.AntPathRequestMatcher - Checking match of request : '/services'; against '/services/**'
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.access.intercept.FilterSecurityInterceptor - Secure object: FilterInvocation: URL: /services; Attributes: [permitAll]
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.access.intercept.FilterSecurityInterceptor - Previously Authenticated: org.springframework.security.authentication.UsernamePasswordAuthenticationToken@4e9c2f63: Principal: org.springframework.security.core.userdetails.User@b163eb6b: Username: dy-sgw; Password: [PROTECTED]; Enabled: true; AccountNonExpired: true; credentialsNonExpired: true; AccountNonLocked: true; Not granted any authorities; Credentials: [PROTECTED]; Authenticated: true; Details: org.springframework.security.web.authentication.WebAuthenticationDetails@3bcc: RemoteIpAddress: 192.168.5.31; SessionId: null; Not granted any authorities
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.access.vote.AffirmativeBased - Voter: org.springframework.security.web.access.expression.WebExpressionVoter@6c2757ee, returned: 1
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.access.intercept.FilterSecurityInterceptor - Authorization successful
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.access.intercept.FilterSecurityInterceptor - RunAsManager did not change Authentication object
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.FilterChainProxy - /services reached end of additional filter chain; proceeding with original chain
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.header.writers.HstsHeaderWriter - Not injecting HSTS header since it did not match the requestMatcher org.springframework.security.web.header.writers.HstsHeaderWriter$SecureRequestMatcher@60a08c3c
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.access.ExceptionTranslationFilter - Chain processed normally
2020-07-13 15:15:10 [http-nio-7070-exec-7] DEBUG org.springframework.security.web.context.SecurityContextPersistenceFilter - SecurityContextHolder now cleared, as request processing completed
```



### AuthenticationEntryPoint

用来解决匿名用户访问无权限资源时异常，例如：http basic请求**无authorization请求头**，实现类BasicAuthenticationEntryPoint会写入WWW-Authenticate", "Basic realm=\"Realm"响应头，要求浏览器提供输入用户名和密码的alert页面，BasicAuthenticationEntryPoint是通过由HttpSecurity.httpBasic()方法调用HttpBasicConfigurer类完成注册BasicAuthenticationEntryPoint。

```java
	void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException;
```

### AccessDeineHandler

 用来解决认证过的用户访问无权限资源时的异常

## 例子

### 除了/services请求,其它请求都必须先认证

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity httpSecurity) throws Exception {
		// @formatter:off
        httpSecurity.csrf().disable();
		httpSecurity.authorizeRequests().
		antMatchers("/services/**").permitAll().
		antMatchers("/**").authenticated().and().httpBasic();
		// @formater:on
	}
}
```

### /services请求允许，/actuator验证身份，其他的禁止

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity httpSecurity) throws Exception {
		// actuator访问需要basic认证,并且禁用CSRF防御
		//@formatter:off
		httpSecurity.authorizeRequests()
		.antMatchers("/actuator/**").authenticated()
		.antMatchers("/services/**").permitAll()
		.antMatchers("/**").denyAll()
		.and().httpBasic()
		.and().csrf().disable();
		 //@formatter:on
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

