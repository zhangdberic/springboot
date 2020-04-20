# spring boot + rabbitmq

## 1.配置

### 1.1 pom.xml

```xml
		<!-- spring boot rabbitmq  -->
		 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

### 1.2 配置属性说明

```yaml
spring: 
  # rabbitmq
  rabbitmq:
    addresses: 192.168.0.250:5672,192.168.0.251:5672
    virtual-host: /
    username: admin
    password: Rabbitmq-401
    connection-timeout: 1000
    cache:
      connection:
        size: 2
        mode: connection    
      channel:
        checkout-timeout: 1000
        size: 2
```

[Spring boot Rabbitmq配置好文章](https://www.jianshu.com/p/2c2a7cfdd38a)

**addresses**：设置了rabbitmq的地址、端口，集群部署的情况下可填写多个，“,”分隔。

**username**：设置rabbitmq的用户名。

**password**：设置rabbitmq的用户密码。

**virtualHost**：设置virtualHost。

**cache.connection.mode**：设置缓存模式，共有两种，\*CHANNEL\*和*CONNECTION*模式。

*CHANNEL*模式，程序运行期间ConnectionFactory会维护着一个Connection，所有的操作都会使用这个Connection，但一个Connection中可以有多个Channel，操作rabbitmq之前都必须先获取到一个Channel，否则就会阻塞（可以通过checkout-timeout设置等待时间），这些Channel会被缓存（缓存的数量可以通过cache.channel.size设置）；

*CONNECTION*模式，这个模式下允许创建多个Connection，会缓存一定数量的Connection，每个Connection中同样会缓存一些Channel，除了可以有多个Connection，其它都跟CHANNEL模式一样。

这里的Connection和Channel是spring-amqp中的概念，并非rabbitmq中的概念，官方文档对Connection和Channel有这样的描述：

> Sharing of the connection is possible since the "unit of work" for messaging with AMQP is actually a "channel" (in some ways, this is similar to the relationship between a Connection and a Session in JMS).

关于*CONNECTION*模式中，可以存在多个Connection的使用场景，官方文档的描述：

> The use of separate connections might be useful in some environments, such as consuming from an HA cluster, in conjunction with a load balancer, to connect to different cluster members.

**cache.channel.size**：设置每个Connection中（注意是每个Connection）可以缓存的Channel数量，注意只是缓存的Channel数量，不是Channel的数量上限，操作rabbitmq之前（send/receive message等）要先获取到一个Channel，获取Channel时会先从缓存中找闲置的Channel，如果没有则创建新的Channel，当Channel数量大于缓存数量时，多出来没法放进缓存的会被关闭。

注意，改变这个值不会影响已经存在的Connection，只影响之后创建的Connection。

**cache.channel.checkoutTimeout**：当这个值大于0时，*channelCacheSize*不仅是缓存数量，同时也会变成数量上限，从缓存获取不到可用的Channel时（也就是说，目前缓冲中所有的Channel都已经被借出去，也就是正在被使用，还没有被还回来），不会创建新的Channel，会等待这个值设置的毫秒数，到时间仍然获取不到可用的Channel会抛出AmqpTimeoutException异常。

同时，在*CONNECTION*模式，这个值也会影响获取Connection的等待时间，超时获取不到Connection也会抛出AmqpTimeoutException异常。

**注意：**这里有个问题，如果使用spring的bean来定义queue、exchange、routingkey如下例子，则不能使用connection缓存模式（不能创建，并且有启动警告： RabbitAdmin auto declaration is not supported with CacheMode.CONNECTION）。你可以先使用channel缓存模式来定义（创建）queue、exchange、routingkey（使用channel模式先启动一下spring boot），再改回到connection缓存模式。

## 2.消息发送者

### 2.1 使用spring来定义消息

如下，创建了：

```java
@Configuration
public class BackupRabbitmqConfiguration {

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
	public Binding fssFileBakRoutingKey(Queue fssFileBakQueue, DirectExchange fssFileBakExchange) {
		// 绑定queue和exchange到routingKey
		return BindingBuilder.bind(fssFileBakQueue).to(fssFileBakExchange)
				.with("fssfilebakrk");
	}

}
```

### 2.2  发送消息

#### 2.2.1 靠近底层的发送消息方式

1.自定义消息属性;

2.自己序列化发送对象;

3.发送byte[]数据;

```java
	@Autowired
	private RabbitTemplate rabbitTemplate;

    public void send(){
		MessageProperties messageProperties = new MessageProperties();
		messageProperties.setAppId("store_bak");
		messageProperties.setContentEncoding("UTF-8");
		messageProperties.setContentType("application/json");
		messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
		String metadataJson = this.metadataJsonSerializer.serialize(uploadMetadata);
		byte[] body = metadataJson.getBytes();        
   	    this.rabbitTemplate.send("ex.hamirrored.fssfilebak","fssfilebakrk", new Message(body,messageProperties));
    }
```

#### 2.2.2 上层发送消息方式

1.声明对象序列化器;

2.直接发送对象,RabbitTemplate自动序列化;

```java
@Configuration
public class BeanConfiguration {

	@Bean
	public RabbitListenerContainerFactory<?> rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
		SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
		factory.setConnectionFactory(connectionFactory);
		// 配置序列化器
		factory.setMessageConverter(new Jackson2JsonMessageConverter());
		return factory;
	}
}
```

```java
	@Autowired
	private RabbitTemplate rabbitTemplate;

    public void send(){
      this.rabbitTemplate.convertAndSend("ex.hamirrored.fssfilebak","fssfilebakrk", uploadMetadata);
    }
```

#### 2.2.3 RabbitTemplate

正常情况下，你可以直接在spring bean中声明@Autowired RabbitTemplate rabbitTemplate; 来使用RabbitTemplate。你也可以自定义RabbitTemplate属性，例如：为RabbitTemplate配置routingKey和queueName属性，再使用这个RabbitTemplate发生消息的时候，就无须再指定发送的routingKey和queueName了。例如：

```java
@Configuration
public class BeanConfiguration {

   @Bean
   public RabbitTemplate fssBakRabbitTemplate(){
      RabbitTemplate rabbitTemplate = new RabbitTemplate();
      rabbitTemplate.setExchange("ex.hamirrored.fssfilebak");
      rabbitTemplate.setRoutingKey("fssfilebakrk");
      return rabbitTemplate;
   }
}

```

```java
@Component
public class Sender(){
	@Autowired
	private RabbitTemplate fssBakRabbitTemplate;

    public void send(){
		MessageProperties messageProperties = new MessageProperties();
		messageProperties.setAppId("store_bak");
		messageProperties.setContentEncoding("UTF-8");
		messageProperties.setContentType("application/json");
		messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
		String metadataJson = this.metadataJsonSerializer.serialize(this.getUploadMetadata(fileAttribute));
		byte[] body = metadataJson.getBytes();        
   	    this.rabbitTemplate.send(new Message(body,messageProperties));
    }    
}
```



### 2.3 接收消息

#### 2.3.1 监听方法

声明spring bean的某个方法为Rabbit监听器方法。在方法上声明@RabbitListener源注释，并配置相关属性，这里的属性可以通过配置属性中获取到，例如：${xxxx}。

@RabbitListener声明这个方法为监听器方法，负责接收Rabbitmq消息；

@QueueBinding绑定exchange到queue，并声明routingKey；

@Queue声明队列和队列属性；

@Exchange声明交换机和属性；

@PayLoad声明被消息反序列化后的对象；

@Header声明要使用的消息头值，例如：这里的DELIVERY_TAG；

下面的例子，是一个手工确认消息的实现；

```java
@Component
public class DfssFssActionListener {

	@RabbitListener(bindings = @QueueBinding(
	        value = @Queue(value = "${fss.backupRabbitmq.queueName}", durable = "true"),
	        exchange = @Exchange(value = "${fss.backupRabbitmq.exchangeName}", durable = "true"), 
	        key = "${fss.backupRabbitmq.routingKey}")
	)
	public void onMessage(@Payload Map<String,String> fssFileBackMap, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, Channel channel) throws Exception {
		//channel.basicAck(deliveryTag,false);
		channel.basicReject(deliveryTag, true); // 拒绝,重新排队到队列尾部
	}

}
```

#### 2.3.2 手工确认消息

```java
channel.basicAck(deliveryTag,false);
```

deliveryTag是一个唯一的消息序号(例如：1,2,3,4)，false只确认当前的这个消息，true则确认这个deliveryTag序号之前的所有消息。

#### 2.3.3 手工拒绝消息

```java
channel.basicReject(deliveryTag, true);
```

deliveryTag是一个唯一的消息序号(例如：1,2,3,4)，true则重新排队这个消息到队列尾部，false则丢弃这个消息。

```
channel.basicNack(deliveryTag, true, false);
```







## 好文章

https://www.jianshu.com/p/2c2a7cfdd38a

http://www.imooc.com/article/274470

https://www.cnblogs.com/wangjing666/p/10038444.html

https://blog.csdn.net/weixin_44106334/article/details/104643585

https://www.cnblogs.com/piaolingzxh/p/5448927.html