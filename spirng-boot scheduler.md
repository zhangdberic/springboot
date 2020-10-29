# spring boot Scheduler



## @EnableScheduling

spring boot启动类加入@EnableScheduling源注释

```java
@SpringBootApplication
@EnableScheduling
public class ElicenseApplication {

	public static void main(String[] args) {
		SpringApplication.run(ElicenseApplication.class, args);
	}

}
```



## @Scheduled

@Scheduled可实现的定时任务种类很多，总有一种适合你。可以上网查看一下资料，下面这个文章很好：https://www.jianshu.com/p/1defb0f22ed1

### 每隔xxxx毫秒执行一次

@Scheduled声明的方法，就要要定时执行(调动)的方法，例如：

这里的初始延时和间隔时间，都使用了外部配置形式。

```java
@Component
public class SyncDataScheduler {
	@Scheduled(initialDelayString = "${elicense.syncdata.initialDelay}", fixedRateString = "${elicense.syncdata.fixedRate}")
	public void task() {
        System.out.println("123");
    }
}
```

## 多任务并发执行

@Scheduled声明的方法，默认是串行执行的，即使你多个方法上声明@Scheduled并且是一个执行时间，其也是串行执行。因为其默认使用newSingleThreadExecutor来创建线程池(单线程scheduling-1)来执行到期的方法，当前多个方法同时到期获取某个方法阻塞的时候，就会造成方法执行时间不准确了。

解决方法，基于线程池来实现：

注意：下面的taskScheduler方法创建的spring bean，其就是任务线程池对象。

```java
@Component
@EnableScheduling
public class MyScheduled {

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        taskScheduler.setPoolSize(50);
        return taskScheduler;
    }

    @Scheduled(cron = "0/5 * * * * ?")
    public void execute1(){
        String curName = Thread.currentThread().getName() ;
        System.out.println("当前时间:"+LocalDateTime.now()+"  任务execute1对应的线程名: "+curName);
        try {
            Thread.sleep(1000);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Scheduled(cron = "0/5 * * * * ?")
    public void execute2(){

        String curName = Thread.currentThread().getName() ;
        System.out.println("当前时间:"+LocalDateTime.now()+"  任务execute2对应的线程名: "+curName);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

