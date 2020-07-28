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



## 3.生产环境devtools

使用maven的spring boot原生插件spring-boot-maven-plugin编译和打包后其会自动去掉devtools包，使用命令：jar tvf sgw-manager-1.0.1.jar |grep dev查看不到任何devtools包，因此运行maven打包后的jar文件无须再关注devtools。测试环境、学习环境、生产环境也无须再有任何devtools的配置了。

