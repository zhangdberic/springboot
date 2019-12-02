spring-boot-devtools实现springboot项目热部署  
==========  

## 1.pom.xml

```xml
<!-- spring boot devtools -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>

<plugins>
	<!-- 引入spring-boot的maven插件 -->
	<plugin>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-maven-plugin</artifactId>
		<configuration>
			<fork>true</fork>
		</configuration>
	</plugin>
</plugins>
```

注意：spring-boot-maven-plugin插件要加入<configuration><fork>true</fork></configuration>配置。



## 2.application.yml

```yaml
spring:  
  # devtools
  devtools: 
    restart: 
      enabled: false  
```

注意：这里spring.devtools.restart.enabled=false，可以实现调试模式下，代码修改马上生效（局限在方法内代码），而不用整个项目重启（重启tomcat）。如果设置为true，则重启整个项目。

