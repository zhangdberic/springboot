# spring-boot 日志

spring boot默认使用logback的实现，你可以通过基于yaml的方式类配置log，但yaml只提供了基本的配置。

## yaml配置

### SQL日志

```yaml
# 日志    
logging:
  level:
    root: info
    org.hibernate.SQL: debug
    org.hibernate.type: trace
    org.hibernate.type.BasicTypeRegistry: error
    org.springframework.orm.jpa.JpaTransactionManager: debug      
    z1.web.tool.debug: DEBUG

```

### apache http client日志

```yaml
# 日志    
logging:
  level:
    root: INFO 
    org.apache.http: DEBUG 
```

### tomcat日志

```yaml
logging:
  level:
    root: WARN
    org.apache.tomcat: WARN
    org.apache.catalina: WARN
```



## spring-logback.xml

### 使用

1.在bootstrap.yml文件中声明spring.application.name属性，不要在application.yml中声明否则日志文件名生成错误。

2.在src/main/resource创建spring-logback.xml文件，内容如下xml(通用)：

3.设置jvm启动属性，-Dloghome=xxx设置日志文件存放的目录，例如：$HOME/logs，默认会设置为程序运行位置的logs目录。

### 生成的文件名

一共会生成4种日志：

appname_info.log

appname_error.log

appname_info.yyyy-MM-dd.i.log

appname_error.yyyy-MM-dd.i.log

info字样是日志级别>=info的日志文件,error字样是日志级别>=error的日志文件.

带日期的都是归档日志文件，不带日期的是当前正在使用的日志文件。

归档日志文件，基于时间和文件大小生成滚动日志，格式为：dy-config_info.2020-06-06.0.log，dy-config为应用名称(spring.application.name)，.2020-06-06为日志生成日期，后名的0为序号(每天都从0开始)，默认一个日志文件的大小为50MB超出了则日志切换，序号加1。

### xml配置说明

spring-logback.xml是logback.xml的springboot版本，其加入了springboot专有的一些属性和配置，例如：

springProperty，可以获取到springboot的属性，例如下面的appname属性，其获取spring.application.name的spring boot属性值。

springProfile，可以根据spring boot的spring.profiles.actives配置(当前运行环境)，来设置不同的日志输出配置。相当于一个if判断语句。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">

	<!-- 属性 -->
	<springProperty scope="context" name="appname" source="spring.application.name" defaultValue="app" />
	<springProperty scope="context" name="loghome" source="logging.loghome" defaultValue="logs"/>
	
	<!-- 控制台输出日志 -->
	<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
		<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
			<pattern>%red(%d{yyyy-MM-dd HH:mm:ss}) %green([%thread]) %highlight(%-5level) %boldMagenta(%logger) - %cyan(%msg%n)</pattern>
			<charset>UTF-8</charset>
		</encoder>
	</appender>
	
	<springProfile name="!dev">
		<!-- INFO级别日志输出到文件 -->
		<appender name="infofile" class="ch.qos.logback.core.rolling.RollingFileAppender">
			<!-- 当前日志文件 -->
			<file>${loghome}/${appname}_info.log</file>
			<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
				<!-- 归档日志文件,文件名:例如:dy-config_info.2020-06-06.0.log -->
				<fileNamePattern>${loghome}/${appname}_info.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
				<!-- 单个日志文件最大50MB,超出则切换,文件保留180天,生成的总文件大小20GB(超出后最旧的文件会被删除)-->
				<maxFileSize>50MB</maxFileSize>
				<maxHistory>180</maxHistory>
				<totalSizeCap>20GB</totalSizeCap>
			</rollingPolicy>
			<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
				<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} -%msg%n</pattern>
				<charset>UTF-8</charset>
			</encoder>
		</appender>
	</springProfile>

	<springProfile name="!dev">
		<!-- ERROR级别日志输出到文件 -->
		<appender name="errorfile" class="ch.qos.logback.core.rolling.RollingFileAppender">
			<!-- 当前日志文件 -->
			<file>${loghome}/${appname}_error.log</file>
			<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
				<!-- 归档日志文件,文件名:例如:dy-config_error.2020-06-06.0.log -->
				<fileNamePattern>${loghome}/${appname}_error.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
				<!-- 单个日志文件最大50MB,超出则切换,文件保留180天,生成的总文件大小20GB(超出后最旧的文件会被删除)-->
				<maxFileSize>50MB</maxFileSize>
				<maxHistory>180</maxHistory>
				<totalSizeCap>20GB</totalSizeCap>
			</rollingPolicy>
	        <!-- 过滤日志，只输出error等级的日志-->
	        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
	            <level>ERROR</level>
	        </filter>
			<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
				<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} -%msg%n</pattern>
				<charset>UTF-8</charset>
			</encoder>
		</appender>
	</springProfile>
				
	<!-- 开发环境 -->
	<springProfile name="dev">
		<root level="INFO">
			<appender-ref ref="console"/>
		</root>	
	</springProfile>
	
	<!-- 测试、学习、生产环境 -->	
	<springProfile name="!dev">
		<root level="INFO">
			<appender-ref ref="infofile"/>
			<appender-ref ref="errorfile"/>
		</root>
	</springProfile>
	
</configuration>

```

### eclipse带颜色日志输出

eclipse原生不支持带颜色的日志输出，需要如下两步：

1.安装插件：ANSI Escape in Console

2.设置spring.output.ansi.enabled=always，例如：

```yaml
spring:
  application:
    name: test
  output:
    ansi:
      enabled: always
```

### yaml配置优先级高于spring-logback.xml

例如：yaml配置了debug级别的包和类，其优先级会高于spring-logback.xml配置。

