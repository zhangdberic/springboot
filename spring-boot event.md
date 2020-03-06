# 基于Spring的事件框架实现的事件处理

# 1.定义一个事件

例如：定义了一个上传事件（UploadEvent）

```java
import org.springframework.context.ApplicationEvent;

@SuppressWarnings("serial")
public class UploadEvent extends ApplicationEvent {

	// List<FileAttribute> fileAttributes
	public UploadEvent(Object source) {
		super(source);
	}

	@SuppressWarnings("unchecked")
	public List<FileAttribute> getFileAttributes() {
		return (List<FileAttribute>) this.getSource();
	}

}
```



## 2.定义一个监听器

实现了两个接口：

1. ApplicationListener<UploadEvent>，监听器，负责监听UploadEvent事件。
2. Ordered接口定义了，ApplicationListener<UploadEvent>监听器执行的优先级（数小则优先）。

如果要实现异步操作，则可以在onApplicationEvent(UploadEvent event) 方法上，声明源注释：@Async

```java
@Component
public class UploadBackupListener implements  ApplicationListener<UploadEvent>, Ordered {

	@Override
	public int getOrder() {
		return 3;
	}

	@Override
	@Async
	public void onApplicationEvent(UploadEvent event) { 
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
				this.rabbitTemplate.send(this.fssConfig.getBackupRabbitmqExchangeName(), this.fssConfig.getBackupRabbitmqRoutingKey(),
						new Message(body, messageProperties));
			}
		}
	}

```

## 3.触发(发布事件)

直接定义属性ApplicationEventPublisher，spring启动的时候，会自动注入。

publishEvent(new UploadEvent(fileAttributes));，发布事件或说成触发事件。

```java
	@Autowired
	private ApplicationEventPublisher applicationEventPublisher;

	@PostMapping(value = "/upload", produces = "application/json;charset=UTF-8")
	public String[] upload(HttpServletRequest request) throws ServiceException {
        			// 触发上传事件
			this.applicationEventPublisher.publishEvent(new UploadEvent(fileAttributes));
    }
```

