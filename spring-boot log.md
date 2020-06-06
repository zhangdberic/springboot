# spring-boot 日志

spring boot默认使用logback的实现，并且提供了基本的日志输出支持，你可以通过基于yaml的方式类配置log，但yaml只提供了基本的配置。

## yaml配置



## spring-logback.xml

### 使用

1.在src/main/resource创建spring-logback.xml文件，内容如下xml(通用)：

2.设置jvm启动属性，-Dloghome=xxx，设置日志文件存放的目录，例如：$HOME/logs

### xml配置说明

spring-logback.xml是logback.xml的springboot版本，其加入了springboot专有的一些属性和配置，例如：

springProperty，可以获取到springboot的属性，例如下面的appname属性，其获取spring.application.name的spring boot属性值。

springProfile，可以根据spring boot的spring.profiles.actives配置(当前运行环境)，来设置不同的日志输出配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">

	<!-- 属性 -->
	<springProperty scope="context" name="appname" source="spring.application.name" defaultValue="app"/>

	<!-- 控制台输出日志 -->
	<appender name="console" class="ch.qos.logback.core.ConsoleAppender">
		<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
			<pattern>%red(%d{yyyy-MM-dd HH:mm:ss}) %green([%thread]) %highlight(%-5level) %boldMagenta(%logger) - %cyan(%msg%n)</pattern>
			<charset>UTF-8</charset>
		</encoder>
	</appender>
	
	<!-- INFO级别日志输出到文件 -->
	<appender name="infofile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 日志还没滚动前存放的日志文件路径 -->
		<file>${loghome}/${appname}_info.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
			<!-- rollover daily -->
			<fileNamePattern>${loghome}/${appname}_info.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
			<!-- 单个日志文件最大50MB,超出则切换,文件保留180天,生成的总文件大小20G(超出后最旧的文件会被删除)-->
			<maxFileSize>50MB</maxFileSize>
			<maxHistory>180</maxHistory>
			<totalSizeCap>20G</totalSizeCap>
		</rollingPolicy>
		<encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
			<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} -%msg%n</pattern>
			<charset>UTF-8</charset>
		</encoder>
	</appender>
	
	<!-- ERROR级别日志输出到文件 -->
	<appender name="errorfile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 日志还没滚动前存放的日志文件路径 -->
		<file>${loghome}/${appname}_error.log</file>
		<rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
			<!-- rollover daily -->
			<fileNamePattern>${loghome}/${appname}_error.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
			<!-- 单个日志文件最大50MB,超出则切换,文件保留180天,生成的总文件大小20G(超出后最旧的文件会被删除)-->
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

	<!-- 开发环境 -->
	<springProfile name="dev,default">
		<root level="INFO">
			<appender-ref ref="console"/>
		</root>	
	</springProfile>
	
	<!-- 测试、学习、生产环境 -->	
	<springProfile name="test,study,proc">
		<root level="INFO">
			<appender-ref ref="console"/>
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

