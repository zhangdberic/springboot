# sprng boot test

## 1.安装和配置

### 1.1 pom.xml

```xml
		<!-- spring boot test -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
```

### 1.2 简单的测试用例

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Z1AutoConfiguration.class)
@ActiveProfiles("test")
public class AESTest {
	
	@Test
	public void test() {
		AES aes = new AES("123456");
		byte[] data = "黑哥".getBytes();
		byte[] encryptedData = aes.encrypt(data);
		byte[] descryptedData = aes.descrypt(encryptedData);
		Assert.assertArrayEquals(data, descryptedData);
	}

}
```

@SpringBootTest源注释，负责@Configuration声明类的加载。

@ActiveProfiles 指定了环境。

## 2.好文章

https://blog.csdn.net/wo541075754/article/details/88983708



## 3.Spring mvc test

### 3.1 测试一个Controller例子

因为要测试Controller，因此要引入web。

```xml
		<!-- spring boot web -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
```

@SpringBootTest 会自动在classpath下查找@SpringBootApplication声明的类，并加载。

@WebAppConfiguration 测试环境使用，用来表示测试环境使用的ApplicationContext将是WebApplicationContext类型的。

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@WebAppConfiguration
@ActiveProfiles("dev")
public class SecurityControllerTest {
	
	private MockMvc mockMvc;
	
	@Autowired
	private LoginController loginController;

	@Autowired
	private WebApplicationContext webApplicationContext;
	
	@Before
	public void setup() {
		// 实例化方式一
		this.mockMvc = MockMvcBuilders.standaloneSetup(this.loginController).build();
	}
	
	@Test
	public void login() throws Exception {

		this.mockMvc.perform(MockMvcRequestBuilders.post("/login.html")
		.param("loginName", "heige")
		.param("password", "12345678"))
		.andExpect(MockMvcResultMatchers.status().isOk())
		.andExpect(MockMvcResultMatchers.model().hasNoErrors())
		.andExpect(MockMvcResultMatchers.model().attributeExists("user"))
		.andExpect(MockMvcResultMatchers.view().name("index"))
		.andDo(MockMvcResultHandlers.print());
		
	}

}
```

.andExpect(MockMvcResultMatchers.status().isOk())  断言响应码为200；

.andExpect(MockMvcResultMatchers.model().hasNoErrors()) model中没有错误；

.andExpect(MockMvcResultMatchers.model().attributeExists("user")) model中存在user对象；

.andExpect(MockMvcResultMatchers.view().name("index")) 返回的页面是index；

.andDo(MockMvcResultHandlers.print()); 最后打印结果信息；

**上面的测试用例，对应的SecurityController代码：**

```java
@Controller
public class LoginController {

	@Autowired
	private SecurityLogic securityLogic;

	@PostMapping("/login.html")
	public String login(@Validated LoginForm loginForm, BindingResult errors, Model model) {
		if (errors.hasErrors()) {
			return "login";
		}
		User user = this.securityLogic.login(loginForm.getLoginName(), loginForm.getPassword(), errors);
		if (errors.hasErrors()) {
			return "login";
		}
		model.addAttribute("user", user);
		return "index";
	}

}
```

