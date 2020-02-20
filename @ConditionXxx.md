# @ConditionXxx Bean装载条件判断

## 1.验证配置属性值

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ ElementType.TYPE, ElementType.METHOD })
@Documented
@Conditional(OnPropertyCondition.class)
public @interface ConditionalOnProperty {

    String[] value() default {}; //数组，获取对应property名称的值，与name不可同时使用  
  
    String prefix() default "";//property名称的前缀，可有可无  
  
    String[] name() default {};//数组，property完整名称或部分名称（可与prefix组合使用，组成完整的property名称），与value不可同时使用  
  
    String havingValue() default "";//可与name组合使用，比较获取到的属性值与havingValue给定的值是否相同，相同才加载配置  
  
    boolean matchIfMissing() default false;//缺少该property时是否可以加载。如果为true，没有该property也会正常加载；反之报错  
  
    boolean relaxedNames() default true;//是否可以松散匹配，至今不知道怎么使用的  
} 
```

例如：z1.ssdb.pool.enabled的属性值必须为true

```java
@ConditionalOnProperty(value = "z1.ssdb.pool.enabled", havingValue = "true")
```

例如：z1.ssdb.pool.enabled属性必须存在(值无所谓)

```java
@ConditionalOnProperty(value = "z1.ssdb.pool.enabled", matchIfMissing = "false")
```



## 2.Class存在

```java
@ConditionalOnClass(EurekaClientConfig.class)
```

## 3.Spring Bean存在

```java
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
```

## 4.判断系统属性值

```java
 	@Bean("hello")
    @ConditionalOnSystemProperty(key = "user.name", value = "user")
    public String userHello() {
        return "hello user !";
    }

```

