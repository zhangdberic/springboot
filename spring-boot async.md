# spring boot - async

## 1.使用async使用异步

### 1.1 启动类加上@EnableAsync

注意，如果忘加了这个源注释，则即使方法加上了@Async源注释，还是同步执行。

```java
@SpringBootApplication
@EnableAsync
public class FssApplication {

	public static void main(String[] args) {
		SpringApplication.run(FssApplication.class, args);
	}
	
}

```

### 1.2 需要异步执行的方法上加上@Async

如果在类上加上@Async则这个类的所有方法都是异步方法。

```java
@Configuration
public class UploadBackupListener implements EventListener<UploadEvent> {

	@Autowired
	private RabbitTemplate rabbitTemplate;

	@Override
	@Async
	public void tigger(UploadEvent event) {
        this.rabbitTemplate.send("ex.hamirrored.fssfilebak", "fssfilebakrk",
						new Message("123", messageProperties));
    }
```

**注意：**@Async修饰方法所在类，如果实现了接口，则应该使用接口类型注射到其他bean中，例如：某个bean要使用下面的UploadBackupListener，应该使用@Autowired EventListener<UploadEvent> uploadBackupListener，而不是@Autowired UploadBackupListener uploadBackupListener。如果非要基于第2种基于类注射，则需要修改：@EnableAsync(proxyTargetClass=true)，这其实就是Spring的两种代理实现，一种基于jdk interface proxy的接口代理，另一种基于CGLIB的类代理。

### 1.3 调用这个异步方法

```java
@Autowired 
private EventListener<UploadEvent> uploadBackupListener;

public void publish(){
    this.uploadBackupListener.tigger(new UploadEvent("123"));
}
```

## 2. 使用独立的线程池

定义一个独立的线程池bean，返回TaskExecutor接口对象。这个ThreadPoolTaskExecutor可以定义线程池的各种属性。

```java
    @Bean
    public TaskExecutor taskExecutor() {  
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor(); 
        executor.setThreadNamePrefix("Anno-Executor");
        executor.setMaxPoolSize(10);  
 
        // 设置拒绝策略
        executor.setRejectedExecutionHandler(new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                // .....
            }
        });
        // 使用预定义的异常处理类
        // executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
 
        return executor;  
    } 
```

@Async括号中指定上面线程池的bean名。


```java
    @Async("taskExecutor")
	public void tigger(UploadEvent event) {
        this.rabbitTemplate.send("ex.hamirrored.fssfilebak", "fssfilebakrk",
						new Message("123", messageProperties));
    }
```



## FAQ

Action:

Consider injecting the bean as one of its interfaces or forcing the use of CGLib-based proxies by setting proxyTargetClass=true on @EnableAsync and/or @EnableCaching.

@Async修饰方法所在类，如果实现了接口，则应该使用接口类型注射到其他bean中，例如：某个bean要使用下面的UploadBackupListener，应该使用@Autowired EventListener<UploadEvent> uploadBackupListener，而不是@Autowired UploadBackupListener uploadBackupListener。如果非要基于第2种基于类注射，则需要修改：@EnableAsync(proxyTargetClass=true)，这其实就是Spring的两种代理实现，一种基于jdk interface proxy的接口代理，另一种基于CGLIB的类代理。





