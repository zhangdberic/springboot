# spring boot json

## jackson

spring boot mvc默认就已经集成了jackson来支持json序列化。

### pom.xml

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

### yaml配置

默认的jackson配置。

```yaml
spring:
    jackson:
      # 设置属性命名策略,对应jackson下PropertyNamingStrategy中的常量值，SNAKE_CASE-返回的json驼峰式转下划线，json body下划线传到后端自动转驼峰式
      property-naming-strategy: SNAKE_CASE
      # 全局设置@JsonFormat的格式pattern
      date-format: yyyy-MM-dd HH:mm:ss
      # 当地时区
      locale: zh
      # 设置全局时区
      time-zone: GMT+8
      # 常用，全局设置pojo或被@JsonInclude注解的属性的序列化方式
      default-property-inclusion: NON_NULL #不为空的属性才会序列化,具体属性可看JsonInclude.Include
      # 常规默认,枚举类SerializationFeature中的枚举属性为key，值为boolean设置jackson序列化特性,具体key请看SerializationFeature源码
      serialization:
        WRITE_DATES_AS_TIMESTAMPS: true # 返回的java.util.date转换成timestamp
        FAIL_ON_EMPTY_BEANS: true # 对象为空时是否报错，默认true
      # 枚举类DeserializationFeature中的枚举属性为key，值为boolean设置jackson反序列化特性,具体key请看DeserializationFeature源码
      deserialization:
        # 常用,json中含pojo不存在属性时是否失败报错,默认true
        FAIL_ON_UNKNOWN_PROPERTIES: false
      # 枚举类MapperFeature中的枚举属性为key，值为boolean设置jackson ObjectMapper特性
      # ObjectMapper在jackson中负责json的读写、json与pojo的互转、json tree的互转,具体特性请看MapperFeature,常规默认即可
      mapper:
        # 使用getter取代setter探测属性，如类中含getName()但不包含name属性与setName()，传输的vo json格式模板中依旧含name属性
        USE_GETTERS_AS_SETTERS: true #默认false
      # 枚举类JsonParser.Feature枚举类中的枚举属性为key，值为boolean设置jackson JsonParser特性
      # JsonParser在jackson中负责json内容的读取,具体特性请看JsonParser.Feature，一般无需设置默认即可
      parser:
        ALLOW_SINGLE_QUOTES: true # 是否允许出现单引号,默认false
      # 枚举类JsonGenerator.Feature枚举类中的枚举属性为key，值为boolean设置jackson JsonGenerator特性，一般无需设置默认即可
      # JsonGenerator在jackson中负责编写json内容,具体特性请看JsonGenerator.Feature
```

### 使用

#### spring mvc



#### ObjectMapper

spring mvc默认就已经实例化了一个ObjectMapper对象bean，你只需要直接引用就可以了。例如：

```java
	@Autowired
	private ObjectMapper objectMapper;
```



### 格式

#### 日期格式

```
	@JsonFormat(pattern="yyyy年MM月dd日 HH时mm分ss秒",timezone = "GMT+8")
	private Date birthDay;
```

