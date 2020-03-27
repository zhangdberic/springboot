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



# 好文章

http://www.pianshen.com/article/6280414949/