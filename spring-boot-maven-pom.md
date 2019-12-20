# spring boot maven pom.xml

## 1. Spring Boot pom.xml 骨架配置

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.free</groupId>
	<artifactId>z1</artifactId>
	<version>1.0.1</version>
	<name>z1</name>
	<description>common java code</description>

	<!-- 配置属性 -->
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.encoding>UTF-8</maven.compiler.encoding>
		<java.version>1.8</java.version>
	</properties>	

	<!-- spring boot parent -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.11.RELEASE</version>
        <relativePath/>
	</parent>

	<!-- 相关依赖包 -->
	<dependencies>
		<!-- spring boot web，例如：web开发 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!-- spring boot devtools -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<optional>true</optional>
		</dependency>		
	</dependencies>
	
	<build>
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
	</build>			

</project>
```

## 2. 常用的spring boot依赖包

```xml
		<!-- spring boot web -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<!-- spring boot test -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

```

## 3. 第三方依赖包

```xml
		<!--  JDK不支持PKCS7,这里需要引入第三方包 -->
		<dependency>
      		<groupId>org.bouncycastle</groupId>
		    <artifactId>bcprov-ext-jdk16</artifactId>
		    <version>1.46</version>
			<optional>true</optional>
		</dependency>	
```

