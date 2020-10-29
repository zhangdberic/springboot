# spring boot aop

## 安装和配置

### pom.xml

```xml
		<!-- spring boot aop -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
```



### ~~@EnableAspectJAutoProxy~~

注意：在完成了引入AOP依赖包后，不需要去做其他配置。AOP的默认配置属性中，spring.aop.auto属性默认是开启的，也就是说只要引入了AOP依赖后，默认已经增加了@EnableAspectJAutoProxy，不需要在程序主类中增加@EnableAspectJAutoProxy来启用。



## 例子

### 简单的例子

@Aspect，在类头声明，把本类标注为Aspect增强类。

@Pointcount，在方法上声明定义切入点，@Pointcount的值是切入点表达式，注意：这个方法是一个空方法。

@Around，在方法上声明，@Around的值为**切入点定义方法名**，@Around表示环绕通知。

```java
@Aspect
@Component
public class ElicenseLogAop {
	
	@Pointcut("execution(public * cn.dongyuit.elicense.client.protocol_layer.manufacture.ManufactureProtocolLayerClient.*(..)))")
    public void ManufactureProtocolLayerClientAspect(){
    }
	
	@Around("ManufactureProtocolLayerClientAspect()")
	public Object doAroundGame(ProceedingJoinPoint pjp) {
		try {
			System.out.println("execute start");
			Object o = pjp.proceed();
			System.out.println("execute end");
			return o;
		} catch (Throwable e) {
			System.out.println("execute exception");
			throw new RuntimeException(e);
		}
	}
	

}
```

通知可有很多，例如：

```java
@Before("ManufactureProtocolLayerClientAspect()")  // 执行前通知 
@After("ManufactureProtocolLayerClientAspect()")  // 执行后通知(正常返回和异常返回都会执行这个通知) 
@AfterReturning("ManufactureProtocolLayerClientAspect()")  // 执行后通知(正常返回)
@AfterThrowing("ManufactureProtocolLayerClientAspect()") // 异常通知
```