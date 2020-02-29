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

2.需要异步执行的bean在类头和方法上加上@Async

```java
@Async
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

