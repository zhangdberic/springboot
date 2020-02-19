# Spring Boot common-pool2创建对象池

**1.实现PooledObjectFactory<T>接口**

```java
public class PooledSSDBFactory implements PooledObjectFactory<SSDB> {

	@Value("${z1.ssdb.ip}")
	private String ssdbIp;

	@Value("${z1.ssdb.port}")
	private int ssdbPort;

	@Value("${z1.ssdb.pool.validateSSDBTestKey:poolTest}")
	private String validateSSDBTestKey;
	
	@Override
	public PooledObject<SSDB> makeObject() throws Exception {
		SSDB ssdb = new SSDB(this.ssdbIp, this.ssdbPort);
		return new DefaultPooledObject<>(ssdb);
	}

	@Override
	public void destroyObject(PooledObject<SSDB> p) throws Exception {
		p.getObject().close();
	}

	@Override
	public boolean validateObject(PooledObject<SSDB> p) {
		try {
			byte[] testValue = new byte[1];
			testValue[0] = 1;
			p.getObject().set(this.validateSSDBTestKey, testValue);
			return true;
		} catch (Exception e) {
			return false;
		}
	}

	@Override
	public void activateObject(PooledObject<SSDB> p) throws Exception {
	}

	@Override
	public void passivateObject(PooledObject<SSDB> p) throws Exception {
	}

}
```

**2.继承GenericObjectPool<T>类**

```java
public class SSDBObjectPool extends GenericObjectPool<SSDB> {

	public SSDBObjectPool(PooledObjectFactory<SSDB> factory, GenericObjectPoolConfig config,
			AbandonedConfig abandonedConfig) {
		super(factory, config, abandonedConfig);
	}

	public SSDBObjectPool(PooledObjectFactory<SSDB> factory, GenericObjectPoolConfig config) {
		super(factory, config);
	}

	public SSDBObjectPool(PooledObjectFactory<SSDB> factory) {
		super(factory);
	}

}
```

**3.继承GenericObjectPoolConfig类,用于配置类**

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

**4.创建Spring Boot的自动配置类**

```java
@EnableConfigurationProperties(SSDBPoolConfig.class)
@ConditionalOnProperty(value = "z1.ssdb.pool.enabled", havingValue = "true")
@Configuration
public class SSDBPoolAutoConfiguration {

	@Bean
	public PooledSSDBFactory pooledSSDBFactory() {
		return new PooledSSDBFactory();
	}

	@Autowired
	private SSDBPoolConfig ssdbPoolConfig;

	@Bean
	public SSDBObjectPool ssdbPool() {
		PooledSSDBFactory ssdbFactory = pooledSSDBFactory();
		SSDBObjectPool ssdbObjectPool = new SSDBObjectPool(ssdbFactory, this.ssdbPoolConfig);
		this.initPool(ssdbObjectPool, this.ssdbPoolConfig.getInitialSize(), this.ssdbPoolConfig.getMaxIdle());
		return ssdbObjectPool;
	}

	private void initPool(SSDBObjectPool ssdbObjectPool, int initialSize, int maxIdle) {
		if (initialSize <= 0) {
			return;
		}
		int size = Math.min(initialSize, maxIdle);
		for (int i = 0; i < size; i++) {
			try {
				ssdbObjectPool.addObject();
			} catch (Exception ex) {
				throw new RuntimeException("init ssdb pool error.",ex);
			}
		}
	}

}
```

**5.创建EnableSSDBPool源注释**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(SSDBPoolAutoConfiguration.class)
public @interface EnableSSDBPool {
}
```

**6.测试用例**

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = { Z1Config.class)
@ActiveProfiles("test")
@EnableSSDBPool
public class SSDBObjectPoolTest {

	@Autowired
	private SSDBObjectPool ssdbObjectPool;

	@Test
	public void test() throws Exception {
		SSDB ssdb = this.ssdbObjectPool.borrowObject();
		this.ssdbObjectPool.returnObject(ssdb);
		// 可以复用上次还回到池中的ssdb实例
		SSDB ssdb1 = this.ssdbObjectPool.borrowObject();
		this.ssdbObjectPool.returnObject(ssdb1);
		Assert.assertEquals(ssdb, ssdb1);
		Assert.assertEquals(2, this.ssdbObjectPool.getBorrowedCount());
	}

}
```

