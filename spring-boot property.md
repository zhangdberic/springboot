# 配置属性

# 1.配置文件

一般情况下应该是5个配置文件，放在src/main/resources目录如下：

application.yml 默认配置文件，存放公共、每种环境都会用到的配置属性。

application-dev.yml  指定开发环境的配置属性。

application-test.yml 指定测试环境的配置属性。

application-study.yml 指定学习环境的配置属性。

application-proc.yml 指定生产环境的配置属性。

可以在application.yml中通过，如下，指定默认的环境：

```yaml
spring:
  profiles:
    active: dev 
```

或者在重新启动的时候，指定运行环境，例如：

```java
java -jar xxx --spring.profiles.active=dev
```

注意： bootstrap.yml这个配置文件在spring boot环境中是无效的，其只有在spring cloud中才有效，因为spring cloud config要先于加载application.yml前，来指定应用名称、运行环境、配置文件gitlabs位置等。

# 2.专用的配置属性类

## 2.1 配置属性类

```java
@ConfigurationProperties(prefix = SSDBPoolConfig.CONFIG_PREFIX)
public class SSDBPoolConfig extends GenericObjectPoolConfig {
	
	public static final String CONFIG_PREFIX = "z1.ssdb.pool";

    /**
     * 初始化连接数
     */
    private int initialSize = 3;

    public int getInitialSize() {
        return initialSize;
    }

    public void setInitialSize(int initialSize) {
        this.initialSize = initialSize;
    }	
}

```

## 2.2 引用属性配置Bean

```java
@EnableConfigurationProperties(SSDBPoolConfig.class)
@Configuration
public class SSDBPoolAutoConfiguration {
	@Autowired
	private SSDBPoolConfig ssdbPoolConfig;
	
}
```

注意这里不需要为SSDBPoolConfig声明为@Bean，@EnableConfigurationProperties(SSDBPoolConfig.class)源注释会自动让这个SSDBPoolConfig成为一个Spring Bean.

**注意：**@EnableConfigurationProperties和@ConfigurationProperties和配对出现的，@EnableConfigurationProperties启动bean后缀处理，对所有标注@ConfigurationProperties的bean进行属性注入处理。

## 开发公共组件的配置bean

有些属性对象需要被复用，例如：一个SSDB配置属性类，其一个项目可以连接多个SSDB，则就需要多个SSDBProperties类，如果你还使用上面的方式(配置单个属性bean)就不对了，因此下面使用@bean方式来配置属性bean，例如：

```java
@Configuration
@EnableConfigurationProperties
public class SSDBBeanConfiguration {
	
	@Bean
	@ConfigurationProperties(prefix = "z1.ssdb1")
	public SSDBProperties ssdbProperties1() {
		return new SSDBProperties();
	}
    
	@Bean
	@ConfigurationProperties(prefix = "z1.ssdb2")
	public SSDBProperties ssdbProperties2() {
		return new SSDBProperties();
	}    
}
```

```java
@Validated
public class SSDBProperties {

	public static final String DEFAULT_CONFIG_PREFIX = "z1.ssdb";

	/** ip地址 */
	@NotEmpty
	private String ip;
	/** 端口 */
	private int port = 8888;
	/** 超时(毫秒) */
	private int timeout = 1000;
	/** 密码 */
	private String password;
	/** 验证ssdb是否有效key */
	private String validateSsdbKey = "poolTest";

	/** 连接池属性 */
	private GenericObjectPoolConfig<SSDB> pool = new GenericObjectPoolConfig<SSDB>();
}
```



## 2.4 配置加密

配置加密使用jasypt-spring-boot，github地址：https://github.com/ulisesbocchio/jasypt-spring-boot

### 2.4.1 常用的属性加密

#### 加密程序

编写加密程序，对其他项目中需要加密的属性，执行加密获取加密后的值。

**pom.xml**，注意：这里必须使用spring boot 2.1 以上，否则报错。

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>cn.dongyuit</groupId>
  <artifactId>spring-boot-encryptor</artifactId>
  <version>0.0.1</version>
  
  	<!-- 配置属性 -->
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.encoding>UTF-8</maven.compiler.encoding>
		<java.version>1.8</java.version>
		<skipTests>true</skipTests>
	</properties>

	<!-- spring boot parent -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.13.RELEASE</version>
		<relativePath />
	</parent>
	
	<!-- 相关依赖包 -->
	<dependencies>
		<!-- spring boot basic -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<!-- spring boot test -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<!-- 配置文件加密 -->		
		<dependency>
            <groupId>com.github.ulisesbocchio</groupId>
            <artifactId>jasypt-spring-boot-starter</artifactId>
            <version>3.0.2</version>
        </dependency>		
	</dependencies>	
					
</project>
```

**编写加密测试用例**

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class Encryptor {
	@Autowired
    StringEncryptor encryptor;
	
	@Autowired
	private Environment env;

    @Test
    public void getPass() {
        String url = encryptor.encrypt("jdbc:mysql://47.97.192.116:3306/sell?characterEncoding=utf-8&useSSL=false&serverTimezone=GMT%2b8");
        String name = encryptor.encrypt("你的数据库名");
        String password = encryptor.encrypt("你的数据库密码");
        System.out.println(encryptor.decrypt(password));
        System.out.println(this.env.getProperty("test.password"));

        System.out.println(url+"----------------");
        System.out.println(name+"----------------");
        System.out.println(password+"----------------");
        Assert.assertTrue(name.length() > 0);
        Assert.assertTrue(password.length() > 0);
    }
}
```

**执行加密测试用例**

```shell
-Djasypt.encryptor.password=xxxxx
```

使用jvm的-Djasypt.encryptor.password属性，指定密码。所有测试用例上的加密处理，都使用这个密码来加密。

**配置文件属性加密测试**

application.yml文件加密属性如下，对于的测试程序，为上面测试用例的this.env.getProperty("test.password")代码，其可以直接获取到加密前的原文。

```yaml
test:
  password: ENC(dSQdMmG9NegQE0TdIRiLdjspqmNIryLvc60NBCRIG7UXPNDTfT30tH+6Jmrsnpgkt47qhwkUpsyG3yJlIq5zDQ==----------------)
```

#### 配置加密属性

1.使用上面的**加密程序**对属性加密，获取加密后的值。

2.配置属性使用加密值，例如;

```yaml
test:
  password: ENC(dSQdMmG9NegQE0TdIRiLdjspqmNIryLvc60NBCRIG7UXPNDTfT30tH+6Jmrsnpgkt47qhwkUpsyG3yJlIq5zDQ==----------------)
```

这里注意：加密的属性值，使用ENV()进行包裹。

3.启动jvm使用-Djasypt.encryptor.password=xxxxx属性指定解密的密码。



# 3.yaml格式

## 常用配置

```yaml
person:
    lastName: hello
    age: 18
    boss: false
    birth: 2017/12/12
    maps: {k1: v1,k2: 12}
    lists:
      - lisi
      - zhaoliu
    dog:
      name: 小狗
      age: 12
```

```java
@Component
@ConfigurationProperties(prefix = "person")
public class Person {
 
    private String lastName;
    private Integer age;
    private Boolean boss;
    private Date birth;
 
    private Map<String,Object> maps;
    private List<Object> lists;
    private Dog dog;

    ...
}
```

```java
public class Dog {
    int name;
    int age;
    public void setName(String name){
        this.name = name;
    }
    public void setAge(String age){
        this.age = age;
    }
}
```

## 特殊字符转义

字面量：普通的值（数字，字符串，布尔）

k: v：字面直接来写；

字符串默认不用加上单引号或者双引号；

""：双引号；不会转义字符串里面的特殊字符；特殊字符会作为本身想表示的意思

name: "zhangsan \n lisi"：输出；zhangsan 换行 lisi

''：单引号；会转义特殊字符，特殊字符最终只是一个普通的字符串数据

name: ‘zhangsan \n lisi’：输出；zhangsan \n lisi

**注意：因为是转义字符，因为单引号和双引号只适用于字符类型，不能用于其他类型。**

## 数组

### array value

```yaml
fssList: fss1,fss2
```



## List 配置

### list value

```yaml
  dfssIdOperationServiceList:
    - "dfss.get"
    - "dfss.delete"
    - "dfss.thumb_image"
    - "dfss.set_metadata"
    - "dfss.get_metadata"
```

### list bean

```yaml
spring:  
  application:  
    name: cleanHDFSFilesJob-dev  
  
hdfs:  
  clean:  
    hdfsCleanStrategies:  
      - keepDays: 30  
        pathPattern: "/strategy/hello1/(\d{12})"  
      - keepDays: 30  
        pathPattern: "/strategy/hello2/(\d{12})"  
      - keepDays: 30  
        pathPattern: "/strategy/hello3/(\d{12})"  
      - keepDays: 30  
        pathPattern: "/strategy/hello4/(\d{12})" 
```

```java
public class HDFSCleanStrategy {  
    private Integer keepDays;  
    private String pathPattern;  
  
    public HDFSCleanStrategy() {  
        super();  
    }  
  
    public HDFSCleanStrategy(Integer keepDays, String pathPattern) {  
        super();  
        this.keepDays = keepDays;  
        this.pathPattern = pathPattern;  
    }  

    public Integer getKeepDays() {  
        return keepDays;  
    }  

    public String getPathPattern() {  
        return pathPattern;  
    }  
    
    public void setKeepDays(Integer keepDays) {  
        this.keepDays = keepDays;  
    }  

    public void setPathPattern(String pathPattern) {  
        this.pathPattern = pathPattern;  
    }  
  
```

```java
@Component  
@ConfigurationProperties(prefix = "hdfs.clean")  
public class HDFSCleanStrategyConfig {  
  
    private List<HDFSCleanStrategy> hdfsCleanStrategies;  
  
    /** 
     * @return the hdfsCleanStrategies 
     */  
    public List<HDFSCleanStrategy> getHdfsCleanStrategies() {  
        return hdfsCleanStrategies;  
    }  
  
    /** 
     * @param hdfsCleanStrategies 
     *            the hdfsCleanStrategies to set 
     */  
    public void setHdfsCleanStrategies(List<HDFSCleanStrategy> hdfsCleanStrategies) {  
        this.hdfsCleanStrategies = hdfsCleanStrategies;  
    }  
  
    /* 
     * (non-Javadoc) 
     * @see java.lang.Object#toString() 
     */  
    @Override  
    public String toString() {  
        return CommonUtils.toJSONString(this);  
    }  
}  
```

## Map 配置

### map value 配置

```yaml
  appDayReqNumLimitMap: 
    "test": 2000
    "lnyg": 2000
    "tgms": 999999999999
```

### map bean 配置

```yaml
sgw:
  # 自定义的限流器配置
  rateLimiterConfig:
    "tgms": 
      rateTimeunit: 1
      rateNum: 100
      maxPermits: 500     
    "lnyg": 
      rateTimeunit: 1
      rateNum: 10
      maxPermits: 20           
```

```java
@Configuration
@ConfigurationProperties(prefix = SgwConfig.CONFIG_PREFIX)
@RefreshScope
public class SgwConfig {
	
	public static final String CONFIG_PREFIX = "sgw";
	
    	/**　自定义的限流器配置　*/
	private Map<String,RateLimiterConfig> rateLimiterConfig;
    
    	public Map<String, RateLimiterConfig> getRateLimiterConfig() {
		return rateLimiterConfig;
	}

	public void setRateLimiterConfig(Map<String, RateLimiterConfig> rateLimiterConfig) {
		this.rateLimiterConfig = rateLimiterConfig;
	}
```

注意：因为SgwConfig声明了@ConfigurationProperties源注释，spring会使用CGLIB进行代理操作，因此如果你调试在setRateLimiterConfig方法上加断点，rateLimiterConfig参数即使赋值正确也是{}，你只有在通过@Autowired SgwConfig sgwConfig声明，并使用方法内调用this.sgwConfig.getRateLimiterConfig()，才能真正的看到属性值配置是否成功。

# 4.spring-configuration-metadata.json

spring-configuration-metadata.json文件自动生成，spring boot在编译的时候，会自动生成META-INF/spring-configuration-metadata.json，生成这个文件依赖于spring-boot-configuration-processor依赖包，其基本原理：实现了javax.annotation.processing.Processor，在java编译阶段可以回调。回调操作执行时，会扫描类路径下所有的@ConfigurationProperties源注释，获取配置类的属性，自动生成spring-configuration-metadata.json文件。

```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>
```

additional-spring-configuration-metadata.json，手工补充属性元数据配置文件，有些情况上面的自动扫描程序，无法准确的获取配置属性（例如：配置属性在父类中则无法获取到），则需要自己编写这个文件作为spring-configuration-metadata.json的补充，文件格式同spring-configuration-metadata.json，其存放位置：src/main/resources/META-INF/additional-spring-configuration-metadata.json，例如：

```json
{
  "properties": [
        {
            "name": "z1.ssdb.pool.testOnBorrow", 
            "type": "java.lang.String", 
            "description": "借出对象验证", 
            "sourceType": "z1.tool.ssdb.pool.SSDBPoolConfig"
        }, {
            "name": "z1.ssdb.pool.jmxEnabled", 
            "type": "java.lang.String", 
            "description": "jmx是否允许", 
            "sourceType": "z1.tool.ssdb.pool.SSDBPoolConfig"
        }   		
    ],
  "hints": []
}
```

# 5.spring cloud bus属性刷新

spring cloud config bus支持属性在线刷新

**pom.xml**

```xml
		<!-- spring cloud config client -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-config</artifactId>
		</dependency>
		<!-- spring boot actuator -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- spring cloud bus -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-bus-amqp</artifactId>
		</dependency>		

```

这里的spring-cloud-starter-bus-amqp是关键，理解为：基于rabbitmq的订阅机制，当前服务或程序监听某个配置队列，一旦配置队列有值，则说明有人发送了刷新请求。

**bootstramp.yml**

```yaml
spring:
  application:
    name: app_name1
  profiles: dev
  cloud:
    config:
      uri: http://config1:8080
      profile: dev  # 指定从config server配置的git上拉取的文件(例如:sc-sample1service-dev.yml)
      username: user1   # config server的basic认证的user
      password: pass1 # config server的basic认证的password    
```

**application.yml**

```yml
spring:
  # 和spring-cloud-starter-bus-amqp配合,用于/bus/refresh分布式服务属性刷新
  rabbitmq:
    host: rabbitmq
    port: 5672
    username: admin
    password: Rabbitmq-401
    
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
      base-path: /xxxYYYzzz
  server:
    port: 18080   

```

spring.rabbitmq.*，配置rabbitmq连接属性。

management.*，配置/actuator的相关属性，因为刷新属性需要发送请求，默认：/actuator/bus-refresh，这里为了安全起见，设置一个特殊的base-path，例如：上面的/xxxYYYzzz（随机生成32位字符），代替/actuator前缀，并且为actuator请求设置一个特殊的端口。

例如：

```
curl -X POST http://192.168.5.78:18080/xxxYYYzzz/bus-refresh
```

上面的配置，允许所有的actuator请求，如果只需要/bus-refresh请求，则设置为：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```



## 支持在线刷新组件列表

1. spring.datasource.hikari 
2. 



# 6. @Value

## 默认值

如果没有外部配置属性，则使用定义的默认值，@Value("${配置属性key:默认值}")，冒号后面的就是这个属性的默认值。例如：someKey默认值为true

```java
@Value("${some.key:true}")
private boolean someKey;
```

如果默认值设为空，也将会被设置成默认值。

```java
@Value("${some.key:}")
private String stringWithBlankDefaultValue;
```

数组的默认值可以使用逗号分割。

```java
@Value("${some.key:one,two,three}")
private String[] stringArrayWithDefaults;
 
@Value("${some.key:1,2,3}")
private int[] intArrayWithDefaults;
```

# 好文章

http://www.pianshen.com/article/6280414949/