# Spring boot actuator

## 接收请求的地址和端口

```yaml
management:
  server:
    address: 127.0.0.1
    port: 18080  
```

注意：actuator开放，应该设置一个独立的端口为actuator访问服务，不应使用和业务服务同一个端口，从安全角度考虑也是非常有必要的。

## 设置端点(功能)

include: "*"，设置所有的功能都开发，你可以设置：include: "env,beans,health"

```yaml
management:
  endpoints:
    web:
      exposure:
        include:
        - "*"
```

## 查看所有的actuator资源

GET请求

```
http://192.168.5.31:27070/actuator
```

## 健康检查显示详细信息

GET请求

```
http://192.168.5.31:27070/actuator/health
```

```yaml
# actuator       
management:
  endpoint:
    health:
      show-details: always          

```

## 查看应用配置信息

GET请求

```
http://192.168.5.31:27070/actuator/configprops
```

## 刷新配置

POST请求

```
curl -u dy-sgw:12345678 -X POST http://192.168.5.31:27070/actuator/bus-refresh
```

使用config中心节点刷新某个服务或应用

POST请求

例如：最后的sgw为应用或者服务id，29000端口为spring cloud config的actuator的端口

```
curl -u dy-config:12345678 -X POST http://192.168.5.76:29000/actuator/bus-refresh/sgw
```



## actuator安全控制

加入security

```xml
		<!-- spring security -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>		
```

yaml

```yaml
spring: 
  security:
    user:
      name: dy-sgw
      password: 12345678
```

加入安全控制代码

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity httpSecurity) throws Exception {
		// @formatter:off
		httpSecurity.authorizeRequests().antMatchers("/actuator/**").authenticated().and().httpBasic();
		// @formater:on
	}
}
```



## 安全shutdown

### application.yml

```yaml
# security shutdown      
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include:
        - "*"
      base-path: /cH8DxFOH8Wme4XHl
  server:
    address: 127.0.0.1
    port: 18080      
```

### 本机发送post请求关键

```
/usr/bin/curl -X POST http://127.0.0.1:18080/cH8DxFOH8Wme4XHl/shutdown
```

## 健康检查

### 关闭redis健康检查

```yaml
management:
  health:
    redis:
      enabled: false
```

