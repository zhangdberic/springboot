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

### 1.2 application.java

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
@Query(value="select u from User u where u.loginName = %:loginName")
public List<User> findByloginName(@Param("loginName") String loginName);
```



### 4.3 @Modifying

在原@Query上再加上一个@Modifying就是修改数据操作了。

```java
@Modifying
@Query(value="update User u set u.loginName=:newName where u.loginName = %:loginName")
public int findByUuidOrAge(@Param("loginName") String loginName,@Param("newName") String
newName);
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

```
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

## 6.验证

### 6.1 错误检查不应依赖于异常

业务方法内部正常情况下不应该抛出异常，不应该依赖于异常来完成验证和检查，例如：唯一性检查，你不应该依赖于cache这个异常来判断是否出现了重复数据，你应该使用一条sql来检查是否已经存储重复项。例如：

错误的唯一性检查例子：

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

正确的例子：

```java
@Service
@Transactional
public class AppInfoLogic {
	public AppInfo add(AppInfo appInfo, Errors errors) {
		// 验证新输入appkey是否已经存在
		if (this.appInfoRepository.findByAppkey(appInfo.getAppkey()) != null) {
			errors.rejectValue("appkey", "unique", "重复的应用键");
			return null;
		}
    }
}
```

### 6.2 Errors参数

使用spring提供的org.springframework.validation.Errors;对象作为最后一个参数，把业务方法中产生的所有错误都存放到这个errors对象中。这样调用业务方法的程序(例如：controller)可以根据error.hasErrors()来判断是否有错误。即使不是MVC结构也是可以的，因为Errors接口不依赖于mvc相关包。例如：

业务方法(logic)：

```java
@Service
@Transactional
public class AppInfoLogic {
	public AppInfo add(AppInfo appInfo, Errors errors) {
		// 验证新输入appkey是否已经存在
		if (this.appInfoRepository.findByAppkey(appInfo.getAppkey()) != null) {
			errors.rejectValue("appkey", "unique", "重复的应用键");
			return null;
		}
    }
}
```

控制器(controller)：

```java
	@PostMapping("/app/add.action")
	public String addSubmit(@Validated AddAppForm addAppForm, BindingResult errors, Model model) {
		if (errors.hasErrors()) {
			return "app/add";
		}

		this.appInfoLogic.add(this.createAppInfo(addAppForm), errors);
		if (errors.hasErrors()) {
			return "app/add";
		}
		return "app/add_success";
	}
```



# FAQ

No validator could be found for constraint 'javax.validation.constraints.Size' validating type 'java.lang.Integer'

Integer类型的属性，使用@NotEmpty或者NotBlank来限制了，这是不对的，应该使用@NotNull

--------------------------------------------------------------------------------------------------------------------------------------

在logic方法已经对jpaRepository抛出的异常进行了catch处理，并且没有再抛出异常，为什么业务方法能自动回滚实物，例如：

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
```

目前理解为jpa容器再执行数据库操作方法时，会自动记录异常到上下文，即使你catch了异常，因为jpa容器能识别到上下文中的异常，也会抛出并回滚事务。

## 5.好文章

https://www.cnblogs.com/suizhikuo/p/9412825.html

https://docs.jboss.org/hibernate/orm/5.2/userguide/html_single/Hibernate_User_Guide.html

