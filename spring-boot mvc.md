# spring boot mvc

## 1.创建mvc项目

### 1.1 创建准备

**注意：不要使用war创建项目，就使用普通的jar创建项目。**

#### 1.1.1 pom.xml

```xml
		<!-- spring boot web -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
```

### 1.2 src/main/java

例如：sgw项目

cn.dongyuit.sgw(域名+应用简称)

cn.dongyuit.sgw.manager(域名+应用简称+模块)

cn.dongyuit.sgw.manager.controller(控制器)

cn.dongyuit.sgw.manager.controller.form(表单包)

cn.dongyuit.sgw.manager.logic(业务逻辑包)

cn.dongyuit.sgw.manager.domain(域对象包)

cn.dongyuit.sgw.manager.repository(持久化对象包)

cn.dongyuit.sgw.manager.properties(属性配置包)

BeanConfigurer.java(bean声明类)

MvcConfigurer(mvc配置类)

SgwManagerApplication(应用启动类)

#### Controller(控制器)

下面是一个典型的控制器类，其实现了登录控制：

##### @GetMapping方法

**进入页面的方法**：以XxxForm命名（例如：loginForm，进入登录页面操作），方法返回值是一个字符串(视图名)，参数为Model（用于存放model数据，页面显示使用），URL映射@GetMapping源注释，URL后缀使用xxx.action。

```java
@Controller
public class LoginController {

	@Autowired
	private SecurityLogic securityLogic;

	@GetMapping("/login.action")
	public String loginForm(Model model, HttpSession session) {
		session.invalidate();
		model.addAttribute("loginForm", new LoginForm());
		return "login";
	}
}
```

##### @PostMapping方法

**表单提交方法**：以XxxSubmit命令（例如：loginSubmit，提交登录操作），方法返回值是一个字符串(视图名)，参数：@Validated LoginForm loginForm页面表单项(请求参数)绑定的对象，@Validated源注释在绑定后会对参数进行有效性验证（具体下面介绍），BindingResult errors绑定失败错误对象，Model（用于存放model数据，页面显示使用），HttpSession session会话对象，可有可无根据实际业务情况定。

一般情况下：

第1步要执行errors.hasErrors()，先判断表单项绑定到form和form验证是否错误。

第2步执行业务方法调用。

第3步要执行errors.hasError()，验证业务方法执行是否有错误。

第4步存放页面要显示的数据到model对象，并返回视图名。

```java
@Controller
public class LoginController {

	@Autowired
	private SecurityLogic securityLogic;

	@PostMapping("/login.action")
	public String loginSubmit(@Validated LoginForm loginForm, BindingResult errors, Model model, HttpSession session) {
		if (errors.hasErrors()) {
			return "login";
		}
		User user = this.securityLogic.login(loginForm.getLoginName(), loginForm.getPassword(), errors);
		if (errors.hasErrors()) {
			return "login";
		}
		model.addAttribute("user", user);
		session.setAttribute("user", user);
		return "index";
	}

}
```

##### @ModelAttribute

**@ModelAttribute修饰参数名**

默认情况下，表单绑定对象类型名(className)(不是参数名)应该和页面上的#springBind("xxxxForm")相同，否则返回到表单页面，则无法获取到表单对象，因为errors中的objectName为表单类型名(className)与#springBind("xxxxForm")对不上，会出现java.lang.IllegalStateException: Neither BindingResult nor plain target object for bean name 'loginForm' available as request attribute的异常，这是就需要使用@ModelAttribute源注释来手工指定绑定的objectName，例如：这里的Form的className是AddAppForm和页面#springBind("addForm")不同，因此需要使用@ModelAttribute("addForm")指定objectName。

```html
#springBind("addForm")
```

```java
	@PostMapping("/app/add.action")
	public String addSubmit(@ModelAttribute("addForm") @Validated AddAppForm addForm, BindingResult errors, Model model) {
	}

```

##### HttpSession控制方法参数

使用HttpSession作为controller方法的参数。

```java
	@PostMapping("/login.action")
	public String loginSubmit(@Validated LoginForm loginForm, BindingResult errors, Model model, HttpSession session) {
		if (errors.hasErrors()) {
			return "login";
		}
		User user = this.securityLogic.login(loginForm.getLoginName(), loginForm.getPassword(), errors);
		if (errors.hasErrors()) {
			return "login";
		}
		model.addAttribute("user", user);
		session.setAttribute("user", user);
		return "index";
	}
```

##### 拦截器(Interceptor)

编写一个拦截器，例如：

```java
@Component
public class SecurityInterceptor extends HandlerInterceptorAdapter {

	@Autowired
	private SgwManagerProperties sgwManagerProperties;

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		String requestUri = request.getRequestURI();

		// 验证是否为公共的uri
		List<String> publicUris = this.sgwManagerProperties.getSecurity().getPublicUris();
		if (!CollectionUtils.isEmpty(publicUris)) {
			if (publicUris.contains(requestUri)) {
				return true;
			}
		}

		HttpSession session = request.getSession(false);
		if (session == null) {
			return false;
		}

		boolean verified = session.getAttribute("user") != null;
		if(!verified) {
			response.sendRedirect("/login.html");
			return false;
		}
		return true;
	}

}

```

添加到mvc配置中

```java
@Configuration
public class MvcConfigurer extends WebMvcConfigurerAdapter {

	@Autowired
	private SecurityInterceptor securityInterceptor;

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		InterceptorRegistration userRegistration = registry.addInterceptor(this.securityInterceptor);
		userRegistration.addPathPatterns("/**/*.action");
	}

}
```



#### Form(表单)

form对象有一个原则，form对象不应与domain对象有任何关联，例如：不应继承于domain对象，也不能使用domain对象作为属性等，这样可以减少耦合性，让form对象和domain对象各司其职。

例如：修改应用信息表单Form对象

```java
public class ModifyAppForm {
	/** id */
	@NotNull(message = "不能为空")
	private Long id;
	/** app键值 */
	@NotBlank(message = "不能为空")
	@Length(max = 30, message = "输入不正确")
	private String appkey;
	/** 修改密码 */
	private boolean changeSecret;
	/** 秘钥 */
	private String secret;
	/** 名称 */
	@NotBlank(message = "不能为空")
	@Length(max = 60, message = "输入不正确")
	private String name;
	/** 日请求流量限制 */
	@NotNull(message = "不能为空")
	@Range(min = 0, max = 999999999, message = "输入不正确")
	private Integer dayReqNumLimit;
	/** 有效性 */
	private boolean valid;

	public ModifyAppForm() {

	}

	public ModifyAppForm(AppInfo appInfo) {
		this.id = appInfo.getId();
		this.appkey = appInfo.getAppkey();
		this.name = appInfo.getName();
		this.dayReqNumLimit = appInfo.getDayReqNumLimit();
		this.valid = appInfo.getValid();
	}

	public AppInfo toAppInfo() {
		AppInfo appInfo = new AppInfo();
		appInfo.setId(this.id);
		appInfo.setAppkey(this.appkey);
		appInfo.setSecret(this.secret);
		appInfo.setName(this.name);
		appInfo.setDayReqNumLimit(this.dayReqNumLimit);
		appInfo.setValid(this.valid);
		return appInfo;
	}

	public boolean validate(Errors errors) {
		if (this.changeSecret) {
			if (!StringUtils.hasText(this.secret)) {
				errors.rejectValue("secret", "required", "不能为空");
				return false;
			}
			if (this.secret.length() > 30) {
				errors.rejectValue("secret", "exceedLengthLimit", "输入不正确");
				return false;
			}
		}
		return errors.hasErrors();
	}
```

##### toDomain()

参见上面的ModifyAppForm的toAppInfo()方法，可以把当前ModifyAppForm表单对象转换到AppInfo对象。AppInfo对象是一个Domain对象，是业务方法appInfoLogic.update的参数。例如：

注意：modifyAppForm.toAppInfo()操作。

```java
	@PostMapping("/app/modify.action")
	public String modifySubmit(@ModelAttribute("modifyForm") @Validated ModifyAppForm modifyAppForm,
			BindingResult errors, Model model) {
		if (errors.hasErrors()) {
			return "app/modify";
		}
		if (modifyAppForm.validate(errors)) {
			return "app/modify";
		}
		this.appInfoLogic.update(modifyAppForm.toAppInfo(), modifyAppForm.isChangeSecret(), errors);
		if (errors.hasErrors()) {
			return "app/modify";
		}
		return "app/modify_success";
	}
```

##### construct(Domain domain)

参见上面的ModifyAppForm的ModifyAppForm(AppInfo appInfo)构造方法，其可以使用appInfo(Domain)对象来生成ModifyAppForm对象。例如：

注意：new ModifyAppForm(appInfo);操作。

```java
	@GetMapping("/app/modify.action")
	public String modifyForm(@RequestParam(required = true) Long id, Model model) {
		AppInfo appInfo = this.appInfoLogic.getAppInfo(id);
		if (appInfo == null) {
			return "/app/find";
		}
		ModifyAppForm modifyAppForm = new ModifyAppForm(appInfo);
		model.addAttribute("modifyForm", modifyAppForm);
		return "app/modify";
	}
```

##### validate(Errors errors)

对javax.validation声明式验证的补充操作，javax.validation和org.hibernate.validator只能对表单属性进行基本的验证，在进行复杂验证的时候，则力不从心，这里我们为form对象定义一个validate(Errors errors)方法对表单进行补充验证，javax.validation和org.hibernate.validator不能满足的地方，使用这个方法来验证，例如：参见上面的ModifyAppForm的validate(Errors erros)方法，其完成了只有changeSecret在为true的情况下，才需要验证secret项。

##### javax.validation和org.hibernate.validator

spring mvc集成了hibernate的validator，其是javax.validator(接口规范)的一种实现。

好文章：https://www.jianshu.com/p/67d3637493c7

###### 不为空验证

```java
@NotNull(message = "不能为空")
```

验证对象不能为null，等同于object!=null验证

```java
@NotBlank(message = "不能为空")
```

验证字符串不为空白，等同于!StringUtils.hasText(string)验证，注意：只能验证字符串类型，否则报错。

No validator could be found for constraint 'javax.validation.constraints.Size' validating type 'java.lang.Integer'

Integer类型的属性，使用@NotEmpty或者NotBlank来限制了，这是不对的，应该使用@NotNull

```
@NotEmpty(message = "不能为空")
```

验证字符串不为空字符串，等同于!StringUtils.hasLength(string)验证，注意：只能验证字符串类型，否则报错。

No validator could be found for constraint 'javax.validation.constraints.Size' validating type 'java.lang.Integer'

Integer类型的属性，使用@NotEmpty或者NotBlank来限制了，这是不对的，应该使用@NotNull

###### 字符串长度验证

```java
@Length(min = 3, max = 60, message = "输入不正确")
```

min可以省略，省略情况下不验证min，max可以省略，省略情况下不验证max。

###### 数字范围验证

```java
@Range(min = 0, max = 999999999, message = "输入不正确")
```

min可以省略，省略情况下不验证min，max可以省略，省略情况下不验证max。



#### Logic(业务类)

几个原则：

1.表单form不应作为业务方法参数，应该使用domain对象作为参数，减少对mvc的耦合性。

2.业务方法内验证数据的时候，不应使用抛出异常的策略，应该使用Errors对象作为方法参数，验证失败则写入errors对象，控制器(controller)在执行完业务方法后优先使用errors.hasErrors()判断是否有错误。

##### 错误检查不应依赖于异常

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

##### Errors参数

使用spring提供的org.springframework.validation.Errors;对象作为最后一个参数，把业务方法中产生的所有错误都存放(收集)到这个errors对象中。这样调用业务方法的程序(例如：controller)可以根据error.hasErrors()来判断是否有错误。即使不是MVC结构也是可以的，因为Errors接口不依赖于mvc相关包。例如：

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

##### 事务操作

参见：spring-boot respository的事务章节。

#### properties(属性配置包)

设计为层次结构，不同的层次对于的不同的配置类，例如：下面的security属性(SecurityProperties)是安全配置类，其定义为总配置类内的一个属性。

```java
@Configuration
@ConfigurationProperties(prefix = SgwManagerProperties.CONFIG_PREFIX)
@RefreshScope
public class SgwManagerProperties {
	
	public static final String CONFIG_PREFIX = "sgwm";
	
	private SecurityProperties security;

	public SecurityProperties getSecurity() {
		return security;
	}

	public void setSecurity(SecurityProperties security) {
		this.security = security;
	}

}

```

```java
public class SecurityProperties {
	
	private List<String> publicUris;

	public List<String> getPublicUris() {
		return publicUris;
	}

	public void setPublicUris(List<String> publicUris) {
		this.publicUris = publicUris;
	}
}
```

```yaml
# sgwm
sgwm:
  security:
    publicUris:
      - "/login.html"
      - "/logout.html"    
```

#### viewResolver(视图解析器)

##### velocity

velocity视图可以参加：spring boot velocity 项目的README.md文档。

```xml
		<!-- spring boot velocity -->
		<dependency>
			<groupId>cn.dongyuit</groupId>
			<artifactId>spring-boot-velocity</artifactId>
			<version>1.0.4</version>
		</dependency>
```

在application.java中加入@EnableViewVelocity源注释

```java
@SpringBootApplication
@EnableViewVelocity
public class SgwManagerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SgwManagerApplication.class, args);
	}
}
```

### 1.3 src/main/resources

src/main/resource

src/main/resource/static (存放存静态文件，例如：js、css、images等)

src/main/resource/templates(存放视图文件，例如：xxx.html，扩展名上是html文件，其内部是html标签+视图脚本)   

src/main/resource/templates/vm_lib(如果是velocity环境,则存放velocity函数文件)

application.yml 配置文件（具体见，spring boot properties文档）

application-dev.yml 开发环境配置文件

application-test.yml 测试环境配置文件

application-proc.yml 生产环境配置文件

## 2.安装部署

1.加入shutdown支持

注意：修改原/actutor的uri为一个随机的uri，例如：/cH8DaFOH9Wme3XHl，防止外界直接访问。

server.address限制了只有本机可以发送actutor相关请求，这里表示为shutdown请求。

如下配置：如果在本机发送/cH8DaFOH9Wme3XHl/shutdown请求，则会shutodown这个spring boot项目。

```yaml
# security shutdown      
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include:
        - "*"
      base-path: /cH8DaFOH9Wme3XHl
  server:
    address: 127.0.0.1
    port: 18080 
```

```shell
/usr/bin/curl -X POST http://127.0.0.1:18080/cH8DaFOH9Wme3XHl/shutdown
```

2.pom.xml中配置spring-boot-maven-plugin插件

```xml
	<build>
		<plugins>
			<!-- 引入spring-boot的maven插件 -->
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
				<configuration>
					<fork>true</fork>
					<executable>true</executable>
				</configuration>
			</plugin>
			<!-- 单元测试配置插件 -->
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-surefire-plugin</artifactId>
				<configuration>
					<!-- 打包跳过测试 -->
					<skip>true</skip>
				</configuration>
			</plugin>			
		</plugins>
		
	</build>  
```

3.在sgw项目上右键->Run As->Maven install，运行后在target目录下生成sgw-manager-1.0.1.jar。

4.使用**sgw**用户，上传sgw-manager-1.0.1.jar到服务器，上传到/home/sgw目录下。

5.运行sgw，shell命令如下：

**startup.sh**

```shell
#!/bin/bash
echo starting...
java -jar sgw-manager-1.0.1.jar --spring.profiles.active=test > log.file 2>log.error &
```

**注意：--spring.profiles.active=xxx，xxx指定了环境：dev、test、study、proc**

**生产环境startup.sh**

```shell
#!/bin/bash
echo starting...
java -jar sgw-manager-1.0.1.jar --spring.profiles.active=proc -Djasypt.encryptor.password=T^m5 > log.file 2>log.error &
```

**stop.sh**

```shell
#!/bin/bash
/usr/bin/curl -X POST http://127.0.0.1:18080/cH8DaFOH9Wme3XHl/shutdown
sleep 5s

PID=$(ps -ef | grep sgw-manager | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
    echo Application is already stopped
else
    echo kill $PID
    kill $PID
fi

```

第1个/usr/bin/curl -X POST http://127.0.0.1:18080/cH8DaFOH9Wme3XHl/shutdown发送actutor的shutdown请求，这个需要在application.yml配置加入配置，才能安全可用，见上面配置。下面的kill代码是防御性编程，保证如果shutdown请求无法停止项目，则kill直接杀死进程。