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
@EntityScan
@EnableJpaRepositories
@EnableTransactionManagement
public class SgwManagerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SgwManagerApplication.class, args);
	}
}
```

#### @EntityScan

@EntityScan 扫描@Entity，这个源注释不是必须的，你可以通过@EntityScan({"xxx.yyy.zzz"})扫描指定包位置下的@Entity标注类，默认扫描当前类所在的包和子包；

#### @EnableJpaRepositories 

@EnableJpaRepositories 扫描JpaRepository，你可以通过@EnableJpaRepositories({"xxx.yyy.zzz"})指定扫描的某个包位置下的JpaRepository类型接口，默认扫描为当前类所在的包和子包；

你可以通过@EntityScan和@EnableJpaRepositories设置扫描包位置，来扫描和加载某个jar(类路径)下的@Entity和JpaRepository，但不建议这样，建议在**这个jar的某个类上声明@EntityScan和@EnableJpaRepositories**，然后使用@Import导入这个类，这样就会优雅的扫描和加载了。具体可以看Z1的BusinessLogBeanConfiguration类，其上声明了@EntityScan和@EnableJpaRepositories；

#### @EnableTransactionManagement 

@EnableTransactionManagement 开启事务；

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

继承JpaRepository接口，无需声明为@Repository，Application.java的@EnableJpaRepositories的源注释会自动扫描当前包和子包下JpaRepository类型的接口，

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
	/**　计算摘要的盐　*/
	private String salt = "xxxx";

	@Autowired
	private DataDigest dataDigest;

	@Autowired
	private UserRepository userRepository;

	public User login(String loginName, String password) {
		User user = this.userRepository.findByLoginName(loginName);
		BusinessAssert.notNull(user, "loginName.notExist", "用户名不存在.");
		BusinessAssert.isTrue(user.isValid(), "loginName.invalid", "无效的用户.");
		String inputPasswordDigst = this.dataDigest.digestString((loginName + password + this.salt).getBytes());
		BusinessAssert.equals(user.getPassword(), inputPasswordDigst, "password.inputError", "密码错误.");
		return user;
	}
```

## 3.ORM

### 3.1 id生成器

#### jpa原生生成器

#### hibernate原生生成器

在主键field上加入，如下源注释：

```java
	@Id
	@GeneratedValue(generator="使用的生成器名称")
	@GenericGenerator(name="生成器名称", strategy = "生成器策略")
	@Column
	private Long id;
```

@GeneratedValue的generator属性值和@GenericGenerato的name属性值是一一对应的，解释为@GeneratedValue指定了要是用那个生成器。

@GenericGenerator指定了要使用的生成器策略，hibernate提供了，如下：

```
	uuid
    hilo
    assigned 
    identity  
    select
    sequence
    seqhilo
    increment
    foreign
    guid
    uuid.hex
    sequence-identity
```

对于的生成器java类

```java
static {  

  GENERATORS.put("uuid", UUIDHexGenerator.class);  

  GENERATORS.put("hilo", TableHiLoGenerator.class);  

  GENERATORS.put("assigned", Assigned.class);  

  GENERATORS.put("identity", IdentityGenerator.class);  

  GENERATORS.put("select", SelectGenerator.class);  

  GENERATORS.put("sequence", SequenceGenerator.class);  

  GENERATORS.put("seqhilo", SequenceHiLoGenerator.class);  

  GENERATORS.put("increment", IncrementGenerator.class);  

  GENERATORS.put("foreign", ForeignGenerator.class);  

  GENERATORS.put("guid", GUIDGenerator.class);  

  GENERATORS.put("uuid.hex", UUIDHexGenerator.class); //uuid.hex is deprecated  

  GENERATORS.put("sequence-identity", SequenceIdentityGenerator.class);  

} 
```

##### increment生成器

生成器会自动执行sql：select max(id) from table；然后id+1来设置这个id字段。高并发的时候不应使用。

```java
	@Id
	@GeneratedValue(generator="increment_generator")
	@GenericGenerator(name="increment_generator", strategy = "increment")
	private Long id;
```

##### assigned生成器

**不建议使用**,因为会先产生一条select语句，具体见“"4.6 save(Entity)"介绍。

程序来设置这个id值，例如：程序使用uuid来设置这个id值。

```java
	@Id
	@GeneratedValue(generator = "assigned_generator")    
	@GenericGenerator(name = "assigned_generator", strategy = "assigned") 
	@Column(name="log_id")
	private String logId;
```

##### 自定义Id生成器

```java
public class JdkUUIDGenerator implements IdentifierGenerator {

	@Override
	public Serializable generate(SharedSessionContractImplementor session, Object object) throws HibernateException {
		String uuid =  UUID.randomUUID().toString().replaceAll("-", "");
		return uuid;
	}

}
```

```java
@Id
@GeneratedValue(generator = "jdkuuid_generator")
@GenericGenerator(name = "jdkuuid_generator", strategy = "z1.util.hibernate.idgenerator.JdkUUIDGenerator")
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

#### @Entity

##### 动态插入(DynamicInsert)和动态更新(@DynamicUpdate)

```java
@Entity
@DynamicInsert
@DynamicUpdate
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

#### 4.2.2 HQL查询

nativeQuery=false(默认)

```java
@Query(value="select u from User u where u.loginName = :loginName")
public List<User> findByloginName(@Param("loginName") String loginName);
```

注意：sql中的参数名变量前面要加入[:]冒号。

如果使用jdk1.8以上版本，并且编译加入(compile with -parameters)选项，则无需使用@Param("xxxx")来声明，jpa会自动获取到方法参数名。

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

### 4.5 动态条件分页查询(Example+Page)

界面上提供了多个查询条件，用户可以选择性的使用其中一个或者多个查询条件，如果查询条件项有值(notEmpty)则作为查询条件，并且可以指定查询条件的匹配规则(相等 =xxx、全模糊 %xxx%、前模糊 %xxx、后模糊 xxx%)，还可以指定排序。查询的结果是一个分页对象，页面脚本(例如:velocity)可以根据page对象，来显示分页的相关信息。

#### 4.5.1 基于ExampleMatcher实现

ExampleMatcher实现动态查询比较简单，其可以轻松的动态查询，并实现like。但其不够灵活，例如：我想动态插入一段条件、基于in或者or的查询等都无法实现。

例如：

```java
@Service
@Transactional
public class ServiceInfoLogic {

    @Autowired
	private ServiceInfoRepository serviceInfoRepository;

	public Page<ServiceInfo> find(ServiceInfo condition, Integer pageNumber, Integer pageSize) {
		ExampleMatcher matcher = ExampleMatcher.matching()
				.withMatcher("code", ExampleMatcher.GenericPropertyMatchers.contains())
				.withMatcher("name", ExampleMatcher.GenericPropertyMatchers.contains())
				.withMatcher("location", ExampleMatcher.GenericPropertyMatchers.contains());
		Example<ServiceInfo> example = Example.of(condition, matcher);
		Pageable pageable = new PageRequest(pageNumber, pageSize, Sort.Direction.DESC, "createdTime");
		Page<ServiceInfo> page = this.serviceInfoRepository.findAll(example, pageable);
		return page;
	}
}
```

输出的SQL：

```java
select * from ( select serviceinf0_.id as id1_1_, serviceinf0_.code as code2_1_, serviceinf0_.created_time as created_time3_1_, serviceinf0_.ser_desc as ser_desc4_1_, serviceinf0_.location as location5_1_, serviceinf0_.manager as manager6_1_, serviceinf0_.name as name7_1_, serviceinf0_.sortcode as sortcode8_1_, serviceinf0_.type_id as type_id12_1_, serviceinf0_.updated_time as updated_time9_1_, serviceinf0_.valid as valid10_1_, serviceinf0_.version as version11_1_ from service_info serviceinf0_ where serviceinf0_.name like ? escape ? order by serviceinf0_.created_time desc ) where rownum <= ?
     
select count(serviceinf0_.id) as col_0_0_ from service_info serviceinf0_ where serviceinf0_.name like ? escape ?

```

有种情况是不执行count语句，查询结果集不满一页，说明已经是最后一页了。

```java
		if (content.size() != 0 && pageable.getPageSize() > content.size()) {
			return new PageImpl<T>(content, pageable, pageable.getOffset() + content.size());
		}
```

ExampleMatcher.GenericPropertyMatchers

```
DEFAULT (case-sensitive)    firstname = ?0    默认（大小写敏感）
DEFAULT (case-insensitive)    LOWER(firstname) = LOWER(?0)    默认（忽略大小写）
EXACT (case-sensitive)    firstname = ?0    精确匹配（大小写敏感）
EXACT (case-insensitive)    LOWER(firstname) = LOWER(?0)    精确匹配（忽略大小写）
STARTING (case-sensitive)    firstname like ?0 + ‘%’    前缀匹配（大小写敏感）
STARTING (case-insensitive)    LOWER(firstname) like LOWER(?0) + ‘%’    前缀匹配（忽略大小写）
ENDING (case-sensitive)    firstname like ‘%’ + ?0    后缀匹配（大小写敏感）
ENDING (case-insensitive)    LOWER(firstname) like ‘%’ + LOWER(?0)    后缀匹配（忽略大小写）
CONTAINING (case-sensitive)    firstname like ‘%’ + ?0 + ‘%’    模糊查询（大小写敏感）
CONTAINING (case-insensitive)    LOWER(firstname) like ‘%’ + LOWER(?0) + ‘%’    模糊查询（忽略大小写）
```

#### 4.5.2 基于Z1框架的的实现

```java
	public Page<ServiceInfo> find(ServiceInfo serviceInfo, Integer pageNumber, Integer pageSize) {
		Dynamic condition = Dynamic.whereBegin();
		if (!ServiceType.ROOT_ID.equals(serviceInfo.getType().getId())) {
			List<Long> typeIds = this.serviceTypeRepository
					.getServiceTypeCurrentAndAllChildIds(serviceInfo.getType().getId());
			condition.and(condition.in("s.type.id", typeIds));

		}
		if (StringUtils.hasText(serviceInfo.getCode())) {
			condition.addSegment("and", "s.code like :code", "code", "%" + serviceInfo.getCode() + "%");
		}
		if (StringUtils.hasText(serviceInfo.getName())) {
			condition.addSegment("and", "s.name like :name", "name", "%" + serviceInfo.getName() + "%");
		}
		if (StringUtils.hasText(serviceInfo.getLocation())) {
			condition.addSegment("and", "s.location like :location", "location", "%" + serviceInfo.getLocation() + "%");
		}
		String cntHql = "select count(*) from ServiceInfo s #{ #dynamic}";
		String hql = "from ServiceInfo s left join fetch s.type #{ #dynamic} order by s.type.id,s.sortcode";
		return new JpaHibernateHqlTemplate<ServiceInfo>(this.em).findPageByDynamic(cntHql, hql, condition, pageNumber,
				pageSize);
	}
```



### 4.6 save(Entity)

spring data jpa的save方法比较特殊，如果你的entity中@Id标注的属性值不为null(有值)，则理解为merge合并，这样就会先产生一个select语句，这个语句是由merge方法触发的，这里就有一个问题，如果你的Id生成策略是assigned(有外部程序来设置id值)，就一定会走merge方法，也就是一定要先产生一个select语句。

```java
	@Transactional
	public <S extends T> S save(S entity) {

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
```



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

再有，这个类无须使用@Component来声明。EntityManager类型实例em，使用@PersistenceContext源注释声明。

spring jpa容器启动的时候，会把JpaRepository接口方法(jpa容器自动实现)和ServiceTypeEmRepository(手工方法)结合。

```java
public class ServiceTypeRepositoryImpl implements ServiceTypeEmRepository {

	@PersistenceContext
	private EntityManager em;

	@Override
	public List<ServiceType> getServiceTypesTree(String condition) {
		String sql = "select t.id,substr(sys_connect_by_path(t.name,'/'),2) as name,t.sortcode,t.valid as valid from SERVICE_TYPE t "
				+ (StringUtils.hasLength(condition) ? condition : "")
				+ " start with t.id=1 connect by prior t.id = t.parent_id ORDER SIBLINGS BY t.sortcode";
		SqlResultTransformer<ServiceType> sqlResultTransformer = new SqlResultTransformer<ServiceType>();
		sqlResultTransformer.addScalar("id", LongType.INSTANCE);
		sqlResultTransformer.addScalar("name", StringType.INSTANCE);
		sqlResultTransformer.addScalar("sortcode", BigDecimalType.INSTANCE);
		sqlResultTransformer.addScalar("valid", BooleanType.INSTANCE);
		sqlResultTransformer.buildConstructorResultTransformer(ServiceType.class, Long.class,
						String.class, BigDecimal.class, Boolean.class);
		return new JpaHibernateSqlTemplate<ServiceType>(this.em).findByNamedParams(sql, (String[]) null,
				(Object[]) null, sqlResultTransformer);
	}


}
```

### 6.1 SQLQuery

Spring JPA的底层实现是Hibernate，有的时候需要直接操作底层，例如，下面的例子：

```java
	public List<ServiceType> getServiceTypesTree(String condition) {
		String sql = "select t.id,substr(sys_connect_by_path(t.name,'/'),2) as name,t.sortcode,t.valid as valid from SERVICE_TYPE t "
				+ (StringUtils.hasLength(condition) ? condition : "")
				+ " start with t.id=1 connect by prior t.id = t.parent_id ORDER SIBLINGS BY t.sortcode";
		SqlResultTransformer<ServiceType> sqlResultTransformer = new SqlResultTransformer<ServiceType>();
		sqlResultTransformer.addScalar("id", LongType.INSTANCE);
		sqlResultTransformer.addScalar("name", StringType.INSTANCE);
		sqlResultTransformer.addScalar("sortcode", BigDecimalType.INSTANCE);
		sqlResultTransformer.addScalar("valid", BooleanType.INSTANCE);
		sqlResultTransformer.buildConstructorResultTransformer(ServiceType.class, Long.class,
						String.class, BigDecimal.class, Boolean.class);
		return new JpaHibernateSqlTemplate<ServiceType>(this.em).findByNamedParams(sql, (String[]) null,
				(Object[]) null, sqlResultTransformer);
	}
```

如果不使用addScalar指定映射的类型，可以能会报错类型错误等。

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