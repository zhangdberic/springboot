# spring boot repository

## 1.安装和配置

### 1.1 pom.xml

```xml
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>	
```

因为JPA需要连接池和数据库jdbc驱动，因此要需要加入：

```xml
		<!-- oracle jdbc driver ojdbc6.jar -->
		<dependency>
			<groupId>com.oracle</groupId>
			<artifactId>ojdbc6</artifactId>
			<version>11.2.0.4.181016</version>
		</dependency>
					
		<!-- https://mvnrepository.com/artifact/org.apache.commons/commons-dbcp2 -->
		<dependency>
		    <groupId>org.apache.commons</groupId>
		    <artifactId>commons-dbcp2</artifactId>
		    <version>2.7.0</version>
		</dependency>
```

### 1.2 application.yml

```yml
spring:
  # jpa
  jpa:
    database-platform: z1.util.jpa.hibernate.OracleDialect
  # oracle dbcp2
  datasource:
    driver-class-name: oracle.jdbc.driver.OracleDriver
    url: jdbc:oracle:thin:@//192.168.0.250:1521/orcl
    username: xxx
    password: xxxxxxx
    dbcp2:
      initial-size: 1
      min-idle: 1
      max-idle: 8
      test-on-borrow: true
      validation-query: select 1 from dual
      validation-query-timeout: 1000     
```

注意：这里使用的自定义的z1.util.jpa.hibernate.OracleDialect，代码如下：

```java
public class OracleDialect extends Oracle10gDialect {
	public OracleDialect() {
		registerHibernateType(Types.NVARCHAR, StandardBasicTypes.STRING.getName());
	}
}
```



### 1.3 application.java

```java
@SpringBootApplication
@EnableJpaRepositories
@EnableTransactionManagement
public class SgwManagerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SgwManagerApplication.class, args);
	}
}
```

@EnableJpaRepositories 开启JpaRepository

@EnableTransactionManagement 开启事务

## 2.代码部分

### 2.1 创建一个Entity(User)

剩去get、set、hashcode、toString方法。

```java
@Entity
@Table(name = "USER_INFO")
public class User {

	@Id
	@Column
	private Integer id;

	@Column
	private String name;

	@Column(name = "login_name")
	private String loginName;

	@Column
	private String password;

	@Column
	private boolean valid;
}
```

### 2.2 创建JpaRepository接口(UserRepository)

继承JpaRepository接口，spring jpa容器会自动实现这个接口并实例化。

```java
public interface UserRepository extends JpaRepository<User, Long> {
	User findByLoginName(String loginName);
}
```

### 2.3 使用JpaRepository接口(UserRepository)

```java
@Service
@Transactional
public class SecurityLogic {
	@Autowired
	private UserRepository userRepository;
    
	public User login(String loginName, String password, Errors errors) {
		User user = this.userRepository.findByLoginName(loginName);
		if (user == null) {
			errors.rejectValue("loginName", "notExist", "用户名不存在.");
			return null;
		}
		return user;
	}
}
```

## 3.ORM

### 3.1 id生成器

#### increment生成器

increment生成器，默认情况下jpa是不提供的，这里使用了hibernate的原生实现。

@GeneratedValue(generator="gen_increment")的generator指定了要生成器，和@GenericGenerator(name="gen_increment", strategy = "increment")的name属性一一对应，strategy属性指定了使用的策略，这里为increment。

```java
	@Id
	@GeneratedValue(generator="gen_increment")
	@GenericGenerator(name="gen_increment", strategy = "increment")
	@Column
	private Long id;
```

#### ManyToOne(多对一)

在相应的属性上声明@ManyToOne，指明为一个多对一对象，这个属性类必须是一个Entity类。@JoinColumn设置外键字段名。

```java
	/** 类别 */
	@ManyToOne
	@JoinColumn(name = "type_id")	
	private ServiceType type;
```

#### 时间类型

```java
    @Temporal(TemporalType.DATE)
    private Date birth;

    @Temporal(TemporalType.TIMESTAMP)
    private Date createTime;
```



## 4.JpaRepository

### 4.1 方法命名操作

JpaRepository支持接口规范方法名查询。JPA引擎会根据方法命名自动生成对应的sql语句，目前支持的关键字如下：

**注意：适合不复杂的操作，命名不要过长。**

| Keyword      | Sample                 | JPQL snippet                                                |
| ------------ | ---------------------- | ----------------------------------------------------------- |
| IsNotNull    | findByAgeNotNull       | ...  where x.age not null                                   |
| Like         | findByNameLike         | ...  where x.name like ?1                                   |
| NotLike      | findByNameNotLike      | ...  where x.name not like ?1                               |
| StartingWith | findByNameStartingWith | ...  where x.name like ?1(parameter bound with appended %)  |
| EndingWith   | findByNameEndingWith   | ...  where x.name like ?1(parameter bound with prepended %) |
| Containing   | findByNameContaining   | ...  where x.name like ?1(parameter bound wrapped in %)     |
| OrderBy      | findByAgeOrderByName   | ...  where x.age = ?1 order by x.name desc                  |
| Not          | findByNameNot          | ...  where x.name <> ?1                                     |
| In           | findByAgeIn            | ...  where x.age in ?1                                      |
| NotIn        | findByAgeNotIn         | ...  where x.age not in ?1                                  |
| True         | findByActiveTrue       | ...  where x.avtive = true                                  |
| Flase        | findByActiveFalse      | ...  where x.active = false                                 |
| And          | findByNameAndAge       | ...  where x.name = ?1 and x.age = ?2                       |
| Or           | findByNameOrAge        | ...  where x.name = ?1 or x.age = ?2                        |
| Between      | findBtAgeBetween       | ...  where x.age between ?1 and ?2                          |
| LessThan     | findByAgeLessThan      | ...  where x.age  <  ?1                                     |
| GreaterThan  | findByAgeGreaterThan   | ...  where x.age > ?1                                       |
| After/Before | ...                    | ...                                                         |
| IsNull       | findByAgeIsNull        | ...  where x.age is null                                    |

**例如：**

```java
public interface UserRepository extends JpaRepository<User, Long> {
	User findByLoginName(String loginName);
}
```

### 4.2 @Query

#### 4.2.1 本地查询原生SQL

nativeQuery=true

```java
public interface UserRepository extends JpaRepository<User, Long> {
    @Query(value="select * from user_info where login_name = ?1" ,nativeQuery=true)
    public List<User> findByLoginName(String loginName);
}
```

#### 4.2.2 HQL查看

nativeQuery=false(默认)

```java
@Query(value="select u from User u where u.loginName = :loginName")
public List<User> findByloginName(@Param("loginName") String loginName);
```

注意：sql中的参数名变量前面要加入[:]冒号。



### 4.3 @Modifying

在原@Query上再加上一个@Modifying就是修改数据操作了。

```java
@Modifying
@Query(value="update User u set u.loginName=:newName where u.loginName = %:loginName")
public int findByUuidOrAge(@Param("loginName") String loginName,@Param("newName") String
newName);
```

### 4.4 flush()

flush()刷新输出sql，正常情况要等到业务方法结束后，spring transaction aop会自动提交事务，但有的时候，需要数据修改的可见性，也就是前面的修改应该对后面操作马上有效，这种情况下需要flush。再有就是db和redis或者rabbitmq需要在同一个事务完成操作，你可以先操作数据库然后flush()输出sql，如果sql执行失败，则下面的redis或者rabbitmq不会执行，如果redis或者rabbitmq执行失败抛出异常，则db的事务也会回滚。



## 5.事务(Transaction)

### 5.1 标准事务

application.java，标记为@EnableTransactionManagement源注释，开启实物，例如：

```java
@SpringBootApplication
@EnableJpaRepositories
@EnableTransactionManagement
public class SgwManagerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SgwManagerApplication.class, args);
	}
}
```

在logic类或方法上标记事务，@Transaction，例如：

如果在类上使用@Transactional，则外部调用这个类的所有方法都使用事务外壳，否则应该标记在方法上。

```java
@Service
@Transactional
public class AppInfoLogic {
}
```

### 5.2 事务回滚

只有是jpaRepository操作方法，抛出异常，则事务就会回滚，不管你是否catch了这个jpaRepository方法，例如：

只要appInfoRepository.saveAndFlush(appInfo)抛出异常，就一点会回滚事务。即使catch了这个方法也是要回滚的。目前理解为jpa容器再执行数据库操作方法时，会把抛出的异常自动记录到上下文，即使你catch了异常，因为jpa容器能识别到上下文中的异常，也会回滚事务并抛出异常。

```java
@Service
@Transactional
public class AppInfoLogic {
    public AppInfo add(AppInfo appInfo, Errors errors) {
		try {
			this.appInfoRepository.saveAndFlush(appInfo);
		} catch (Exception uniqueAppKeyException) {
			errors.rejectValue("appkey", "unique", "重复的应用键");
			return null;
		}
    }
 }
```

## 6.EntityManager

有些时候，方法命名和@Query都不能满足要求，例如：需要动态组装sql，这个时候，就需要使用原生的JPA EntityManager来操作数据库了。

JpaRespository使用EntityManager最好按照如下步骤来：

1.声明一个扩展接口，这个接口内的方法，需要调用原生的EntityManager，例如：

```java
public interface ServiceTypeEmRepository {
	public List<ServiceType> getServiceTypesTree(String condition);
}
```

2.继承这个扩展接口，例如：同时继承JpaRepository和上面声明ServiceTypeEmRepository接口：

```java
public interface ServiceTypeRepository extends JpaRepository<ServiceType, Long> ,ServiceTypeEmRepository{
}
```

3.实现这个扩展接口，类名必须是主接口名+Impl（不能扩展接口名+Impl)，这个要注意：

例如：扩展接口实现类的类名ServiceTypeRepositoryImpl，不能是ServiceTypeEmRepositoryImpl，否则报错：

Caused by: org.springframework.data.mapping.PropertyReferenceException: No property getServiceTypesTree found for type ServiceType!

再有，这个类无须使用@Component来声明。

spring jpa容器启动的时候，会把JpaRepository接口方法(jpa容器自动实现)和ServiceTypeEmRepository(手工方法)结合。

```java
public class ServiceTypeRepositoryImpl implements ServiceTypeEmRepository {

	@PersistenceContext
	private EntityManager em;

	@SuppressWarnings("unchecked")
	@Override
	public List<ServiceType> getServiceTypesTree(String condition) {
		String sql = "select t.id,substr(sys_connect_by_path(t.name,'/'),2) as name,t.sortcode,t.valid from SERVICE_TYPE t "
				+ (StringUtils.hasLength(condition) ? condition : "")
				+ " start with t.id=1 connect by prior t.id = t.parent_id ORDER SIBLINGS BY t.sortcode";
		Query query = em.createNativeQuery(sql);
		SQLQuery sqlLQuery = query.unwrap(SQLQuery.class);
		sqlLQuery.addScalar("id", LongType.INSTANCE);
		sqlLQuery.addScalar("name", StringType.INSTANCE);
		sqlLQuery.addScalar("sortcode", BigDecimalType.INSTANCE);
		sqlLQuery.addScalar("valid", BooleanType.INSTANCE);
		sqlLQuery
				.setResultTransformer(new AliasToBeanConstructorResultTransformer(
						org.springframework.util.ClassUtils.getConstructorIfAvailable(ServiceType.class, Long.class,
								String.class, BigDecimal.class, Boolean.class)));
		return query.getResultList();
	}

}
```

#### SQLQuery

Spring JPA的底层实现是Hibernate，有的时候需要直接操作底层，例如，下面的例子：

```java
	public List<ServiceType> getServiceTypesTree(String condition) {
		String sql = "select t.id,substr(sys_connect_by_path(t.name,'/'),2) as name,t.sortcode,t.valid from SERVICE_TYPE t "
				+ (StringUtils.hasLength(condition) ? condition : "")
				+ " start with t.id=1 connect by prior t.id = t.parent_id ORDER SIBLINGS BY t.sortcode";
		Query query = em.createNativeQuery(sql);
		SQLQuery sqlLQuery = query.unwrap(SQLQuery.class);
		sqlLQuery.addScalar("id", LongType.INSTANCE);
		sqlLQuery.addScalar("name", StringType.INSTANCE);
		sqlLQuery.addScalar("sortcode", BigDecimalType.INSTANCE);
		sqlLQuery.addScalar("valid", BooleanType.INSTANCE);
		sqlLQuery
				.setResultTransformer(new AliasToBeanConstructorResultTransformer(
						org.springframework.util.ClassUtils.getConstructorIfAvailable(ServiceType.class, Long.class,
								String.class, BigDecimal.class, Boolean.class)));
		return query.getResultList();
	}
```

如果不使用addScalar指定映射的类型，就会报错如下：

## 5.FAQ

org.hibernate.MappingException: No Dialect mapping for JDBC type: -9

JDBC的类型(-9)，无法映射到java类型，可以有两种解决方法：

1.本地sql，使用sqlLQuery.addScalar("xxx", StandardBasicTypes);手工指定类型。

2.扩展数据库方言类，例如：

```java
public class OracleDialect extends Oracle10gDialect {

	public OracleDialect() {
		registerHibernateType(Types.NVARCHAR, StandardBasicTypes.STRING.getName());
	}

}
```

```yaml
spring:
  jpa:
    database-platform: z1.util.jpa.hibernate.OracleDialect
```



## 6.好文章

https://www.cnblogs.com/suizhikuo/p/9412825.html

https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html

https://blog.csdn.net/moshowgame/article/details/80650641

http://www.seotest.cn/jishu/34389.html

https://cloud.tencent.com/developer/article/1492946

https://www.jianshu.com/p/074d6815a000

https://my.oschina.net/MeiJianMing/blog/1932995