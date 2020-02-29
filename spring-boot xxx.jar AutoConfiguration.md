# 依赖于Spring Boot - 工具项目(z1)

## 工具项目 - Bean配置器

```java
package com.free.z1.config;

@Configuration
@import({ApplicationContextUtils.class})
public class Z1Config {
}
```

工具项目提供自动配置

```properties
# 在资源目录(resources)下新建目录 META-INF
# 在 META-INF 目录下新建文件 spring.factories
# 在文件中添加 org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.free.z1.config.Z1Config
```

第三方Spring Boot项目只要在pom.xml中配置依赖本工具jar包，则会在SpringBoot项目启动时自动配置工具bean。

```xml
<dependency>
    <groupId>com.free</groupId>
    <artifactId>z1</artifactId>
    <version>1.0.1</version>
</dependency>
```

## 工具项目 - 静态获取SpringBean

```java
@Configuration
public class ApplicationContextUtils implements ApplicationContextAware {
    private static volatile boolean initialized;
    private static volatile ApplicationContext applicationContext;
    
    @Override
    public void setApplicationContext(AplicationContext applicationContext) {
        ApplicationContextUtils.applicationContext = applicationContext;
        initialized = true;
    }
    
    private static void requiredInitialized(){
       if(!initialized){
            throw new IllegalStateException("Spring ApplicationContext has not been initialized.");
        }
    }
    
    public static <T> T getBean(Class<T> clazz){
		requiredInitialized();
        return applicationContext.getBean(clazz);
    }
    
    public static <T> T getBean(String name){
		requiredInitialized();
        return applicationContext.getBean(name);
    }
    
}
```

## 工具项目 - 获取配置属性

```java
Environment environment = ApplicationContextUtils.getBean(Environment.class);
```

## 工具项目 - 内部测试

```java
@RunWith(SpringJunit4ClassRunner.class)
@SpringBootTest(classes=Z1Config.class)
@AcitveProfiles("test")
public class XxxTest {
    
    @Autowired
    private Environment environment;
    
    @Test
    public void test(){
    }
    
}
```

