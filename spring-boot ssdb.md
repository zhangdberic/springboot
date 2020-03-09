# spring-boot + ssdb

spring boot 没有集成ssdb，因为毕竟ssdb是一个小众的KV。

ssdb只提供了一个java客户端，没有连接池支持，我们这里要实现一个基于spring boot + 连接池的解决方案。

## 1.spring boot集成ssdb（使用篇）

## 1.1 配置

```yaml
# z1配置  
z1:
  ssdb:
    ip: 192.168.1.1
    port: 6401   
    password: 012345678901234567890123456789ab
    timeout: 100
    pool: 
      enabled: true
      testOnBorrow: true
      jmxEnabled: false
```

这里pool的配置属性，就是org.apache.commons.pool2.impl.GenericObjectPoolConfig类的配置属性。

**注意：**连接池的实现依赖于jakarta common-pool2，如果项目没有要加入这个jar包。

**注意：**这里的配置属性jmxEnabled：false，如果不配置这个或者配置为true，则可以出现jmx重复注册的问题，特别项目中同时存在redis的时候。

## 1.2 应用启动代码

```java
@SpringBootApplication
@EnableSSDB
public class FssApplication {

	public static void main(String[] args) {
		SpringApplication.run(FssApplication.class, args);
	}
	
}

```

## 1.3 ssdbTemplate

```java
@Component
public class SSDBMetadataStorage implements MetadataStorage {

	@Autowired
	private MetadataJsonSerializer metadataJsonSerializer;

	@Autowired
	private SSDBTemplate ssdbTemplate;

	@Override
	public void set(Map<DfssId, Metadata> metadatas) throws IOException {
		Assert.notEmpty(metadatas, "argument[metadatas] must be not empty.");
		List<String> multiSetArgs = new ArrayList<String>(metadatas.size() * 2);
		for (Entry<DfssId, Metadata> metadataEntry : metadatas.entrySet()) {
			String strDfssId = metadataEntry.getKey().toString();
			Metadata metadata = metadataEntry.getValue();
			String metadataJson = this.metadataJsonSerializer.serialize(metadata);
			multiSetArgs.add(strDfssId);
			multiSetArgs.add(metadataJson);
		}

		try {
			this.ssdbTemplate.handle(new SSDBCallback<Void>() {
				@Override
				public Void doInSSDB(SSDB ssdb) throws SSDBException {
					try {
						// multi_set("k1","v1","k2","v2")
						ssdb.multi_set(multiSetArgs.toArray(new String[0]));
					} catch (Exception ex) {
						throw new SSDBException("ssdb execute multi_set method error.", ex);
					}
					return null;
				}
			});
		} catch (SSDBException ex) {
			throw new IOException("ssdb store metdata error.", ex);
		}
	}

	@Override
	public void delete(DfssId... dfssIds) throws IOException {
		Assert.notEmpty(dfssIds, "argument[dfssIds] must be not empty.");
		String[] deleteFssIds = new String[dfssIds.length];
		for (int i = 0; i < dfssIds.length; i++) {
			deleteFssIds[i] = dfssIds[i].toString();
		}
		try {
			this.ssdbTemplate.handle(new SSDBCallback<Void>() {
				@Override
				public Void doInSSDB(SSDB ssdb) throws SSDBException {
					try {
						ssdb.multi_del(StringUtils.arrayToCommaDelimitedString(deleteFssIds));
					} catch (Exception ex) {
						throw new SSDBException("ssdb execute multi_del method error.", ex);
					}
					return null;
				}
			});
		} catch (SSDBException ex) {
			throw new IOException("ssdb delete metdata error.", ex);
		}
	}

}
```



```

```