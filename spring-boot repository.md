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







## 5.好文章

https://www.cnblogs.com/suizhikuo/p/9412825.html



