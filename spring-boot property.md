# 配置属性

# 1.专用的配置属性类

## 1.1 配置属性类

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

## 1.2 引用属性配置Bean

```java
@EnableConfigurationProperties(SSDBPoolConfig.class)
@Configuration
public class SSDBPoolAutoConfiguration {
	@Autowired
	private SSDBPoolConfig ssdbPoolConfig;
	
}
```

注意这里不需要为SSDBPoolConfig声明为@Bean，@EnableConfigurationProperties(SSDBPoolConfig.class)源注释会自动让这个SSDBPoolConfig成为一个Spring Bean.
