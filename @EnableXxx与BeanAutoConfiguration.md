# @EnableXxx+AutoConfiguration

SpringBoot很多组件的开启方式都是@EnableXxxx，例如：@EnableFeign、@EnableZuulProxy等。下面我们使用SSDB做例子，来实现基于@EnableSSDB的自动配置。

## @EnableSSDB的测试用例

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = { Z1Config.class})
@EnableSSDB
@ActiveProfiles("test")
public class SSDBTemplateTest {

	@Autowired
	private SSDBTemplate ssdbTemplate;
	
	@Test
	public void testSetAndGet() throws Exception {

		this.ssdbTemplate.handle(new SSDBCallback<Void>() {
			@Override
			public Void doInSSDB(SSDB ssdb) throws SSDBException {
				try {
					ssdb.set("heige", "黑哥");
					Assert.assertEquals("黑哥", new String(ssdb.get("heige")));
				} catch (Exception ex) {
					throw new SSDBException(ex);
				}
				return null;
			}
		});
	}
}

```

这里我们重点关注：@EnableSSDB源注释

## @EnableSSDB

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(SSDBTemplateAutoConfiguration.class)
public @interface EnableSSDB {
}
```

这里我们重点关注：**@Import(SSDBTemplateAutoConfiguration.class)**，Spring Boot在启动的时候，会获取每个源注释内的@Import源注释，并加载其声明的class，例如：这里的SSDBTemplateAutoConfiguration.class。

## SSDBTemplateAutoConfiguration

```java
@Configuration
@Import(SSDBPoolAutoConfiguration.class)
public class SSDBTemplateAutoConfiguration {

	@Bean("ssdbTemplate")
	@ConditionalOnBean(SSDBPoolAutoConfiguration.class)
	public SSDBTemplate poolSsdbTemplate() {
		return new PoolSSDBTemplate();
	}
	
	@Bean("ssdbTemplate")
	@ConditionalOnMissingBean(SSDBPoolAutoConfiguration.class)
	public SSDBTemplate simpleSSdbTemplate() {
		return new SimpleSSDBTemplate();
	}

}
```

这个类首先声明为了@Configuration，说明了其内的方法用于创建Spring Bean。

@Import(SSDBPoolAutoConfiguration.class)，导入另一个@Configuration声明的类，SSDBPoolAutoConfiguration类用于SSDB连接池的自动配置（下面会介绍）。

类内的两个方法，声明了**创建同一个Spring Bean（@Bean("ssdbTemplate")）**，如下两个源注释，其会根据Spring的上下文中是否有SSDBPoolAutoConfiguration类型的bean，来决定运行的创建方法。

@ConditionalOnBean(SSDBPoolAutoConfiguration.class)，Spring上线文件存在这个类型的bean。

@ConditionalOnMissingBean(SSDBPoolAutoConfiguration.class)，Spring上线文件不存在这个类型的bean。

## SSDBPoolAutoConfiguration

```java
@Configuration
@EnableConfigurationProperties(SSDBPoolConfig.class)
@ConditionalOnProperty(value = "z1.ssdb.pool.enabled", havingValue = "true")
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

这个类首先声明为了@Configuration，说明了其内的方法用于创建Spring Bean。

@EnableConfigurationProperties(SSDBPoolConfig.class)，加载SSDBPoolConfig配置类，使其成为Spring的Bean。其可以直接在类内使用@Autowired来注释这个配置类（SSDBPoolConfig.class）。

**@ConditionalOnProperty(value = "z1.ssdb.pool.enabled", havingValue = "true")**，这个是重点，Spring上下文件根据配置属性z1.ssdb.pool.enabled值是否为true来决定SSDBPoolAutoConfiguration是否执行。如果SSDBPoolAutoConfiguration不执行，其内的所有@Bean声明的方法都不会执行，也就是说ssdbPool()方法不会执行，也就不会创建SSDBObjectPool的SpringBean。





