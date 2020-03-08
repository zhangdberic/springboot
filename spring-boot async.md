# spring boot - async

1.启动类加上@EnableAsync

```java
@SpringBootApplication
@EnableAsync
public class FssApplication {

	public static void main(String[] args) {
		SpringApplication.run(FssApplication.class, args);
	}
	
}

```

2.需要异步执行的方法上加上@Async，如果在类上加上@Async则这个类的所有方法都是异步方法。

**注意：**@Async修饰方法所在类，如果实现了接口，则应该使用接口类型注射到其他bean中，例如：某个bean要使用下面的UploadBackupListener，应该使用@Autowired EventListener<UploadEvent> uploadBackupListener，而不是@Autowired UploadBackupListener uploadBackupListener。如果非要基于第2种基于类注射，则需要修改：@EnableAsync(proxyTargetClass=true)，这其实就是Spring的两种代理实现，一种基于jdk interface proxy的接口代理，另一种基于CGLIB的类代理。

```java
@Configuration
public class UploadBackupListener implements EventListener<UploadEvent> {

	@Bean
	public DirectExchange fssFileBakExchange() {
		// 注册一个 Direct 类型的交换机 默认持久化、非自动删除
		return new DirectExchange("ex.hamirrored.fssfilebak");
	}

	@Bean
	public Queue fssFileBakQueue() {
		// 注册队列
		return new Queue("queue.hamirrored.fssfilebak");
	}

	@Bean
	public Binding fssFileExchangeBinging(Queue fssFileBakQueue, DirectExchange fssFileBakExchange) {
		// 将队列以 info-msg 为绑定键绑定到交换机
		return BindingBuilder.bind(fssFileBakQueue).to(fssFileBakExchange).with("fssfilebakrk");
	}

	@Autowired
	private RabbitTemplate rabbitTemplate;

	@Autowired
	private MetadataJsonSerializer metadataJsonSerializer;

	@Override
	@Async
	public void tigger(UploadEvent event) {
		List<FileAttribute> fileAttributes = event.getFileAttributes();
		if (!CollectionUtils.isEmpty(fileAttributes)) {
			MessageProperties messageProperties = new MessageProperties();
			messageProperties.setAppId("store_bak");
			messageProperties.setContentEncoding("UTF-8");
			messageProperties.setContentType("application/json");
			messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
			for (FileAttribute fileAttribute : fileAttributes) {
				String metadataJson = this.metadataJsonSerializer.serialize(this.getUploadMetadata(fileAttribute));
				byte[] body = metadataJson.getBytes();
				this.rabbitTemplate.send("ex.hamirrored.fssfilebak", "fssfilebakrk",
						new Message(body, messageProperties));
			}
		}
	}

	public Metadata getUploadMetadata(FileAttribute fileAttribute) {
		Metadata metadata = new Metadata();
		metadata.put("id", fileAttribute.getDfssId().toString());
		metadata.put("action", "upload");
		metadata.put(Metadata.FILE_NAME, fileAttribute.getFileName());
		metadata.put(Metadata.CREATED_TIME, fileAttribute.getCreatedTime());
		metadata.put(Metadata.SIZE, String.valueOf(fileAttribute.getSize()));
		return metadata;
	}

}
```



## FAQ

Action:

Consider injecting the bean as one of its interfaces or forcing the use of CGLib-based proxies by setting proxyTargetClass=true on @EnableAsync and/or @EnableCaching.

@Async修饰方法所在类，如果实现了接口，则应该使用接口类型注射到其他bean中，例如：某个bean要使用下面的UploadBackupListener，应该使用@Autowired EventListener<UploadEvent> uploadBackupListener，而不是@Autowired UploadBackupListener uploadBackupListener。如果非要基于第2种基于类注射，则需要修改：@EnableAsync(proxyTargetClass=true)，这其实就是Spring的两种代理实现，一种基于jdk interface proxy的接口代理，另一种基于CGLIB的类代理。





