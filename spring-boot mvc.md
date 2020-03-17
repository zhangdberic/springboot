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

### 1.2 目录结构

### src/main/java

例如：sgw项目

cn.dongyuit.sgw(域名+应用简称)

cn.dongyuit.sgw.manager(域名+应用简称+模块)

cn.dongyuit.sgw.manager.controller(控制器)

cn.dongyuit.sgw.manager.controller.form(表单包)

cn.dongyuit.sgw.manager.logic(业务逻辑包)

cn.dongyuit.sgw.manager.domain(域对象包)

cn.dongyuit.sgw.manager.repository(持久化对象包)

cn.dongyuit.sgw.manager.properties(属性配置包)

BeanDefinition.java(bean声明类)

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

一般情况下，第1步要执行errors.hasErrors()，先判断表单项绑定到form和form验证是否错误。

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



#### Form(表单)

form对象有一个原则，form对象不能和domain对象有任何关联，例如：不应继承于domain对象，也不能使用domain对象作为属性等，这样可以减少耦合性，让form对象和domain对象各司其职。

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

参见上面的ModifyAppForm的toAppInfo()方法，可以把当前ModifyAppForm表单对象转换到AppInfo对象。AppInfo对象是一个Domain对象，是修改业务方法appInfoLogic.update的参数。例如：

注意这里的modifyAppForm.toAppInfo()操作。

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

注意这里的new ModifyAppForm(appInfo);操作。

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

对javax.validation声明式的补充操作，javax.validation和org.hibernate.validator只能对表单属性进行基本的验证，在进行复杂验证的时候，则力不从心，这里我们为form对象定义一个validate(Errors errors)方法对表单进行补充验证，javax.validation和org.hibernate.validator不能满足的地方，使用这个方法来验证，例如：参见上面的ModifyAppForm的validate(Errors erros)方法，其完成了只有changeSecret在为true的情况下，才需要验证secret项。

##### javax.validation和org.hibernate.validator

spring mvc集成了hibernate的validator，其是javax.validator(接口规范)的一种实现。

###### 不为空验证

```java
@NotNull(message = "不能为空")
```

验证对象不能为null，等同于object!=null验证

```java
@NotBlank(message = "不能为空")
```

验证字符串不为空白，等同于!StringUtils.hasText(string)验证，注意：只能验证字符串类型，否则报错。

```
@NotEmpty(message = "不能为空")
```

验证字符串不为空字符串，等同于!StringUtils.hasLength(string)验证，注意：只能验证字符串类型，否则报错。

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

2.业务方法内验证数据的时候，不应使用抛出异常的策略，应该使用Errors作为方法参数，验证失败则写入errors对象，控制器(controller)在执行完业务方法后优先使用errors.hasErrors()判断是否有错误。

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

##### 事务操作

参见：spring-boot respository的事务章节。

#### properties(属性配置包)

设计为层次结构，不同的层次对于的不同的配置类，例如：下面的security属性(SecurityProperties)是安全配置类，其定义了总配置类内的一个属性。

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











Controller中使用会话(Session)

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



拦截器

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



集成velocity2.0

因为高版本的spring boot(大于1.4)已经不再支持velocity，因此需要自己集成。

pom.xml

```xml
		<dependency>
		   <groupId>org.apache.velocity</groupId>
		   <artifactId>velocity-engine-core</artifactId>
		   <version>2.2</version>
		</dependency>
		 
		<dependency>
		   <groupId>org.apache.velocity</groupId>
		   <artifactId>velocity-tools</artifactId>
		   <version>2.0</version>
		</dependency>
```

VelocityViewResolver

```java
@SuppressWarnings("deprecation")
@Configuration
public class VelocityBeanDefinition {

	@Bean
	public VelocityConfig velocityConfig() {
		VelocityConfigurer config = new VelocityConfigurer();
		config.setResourceLoaderPath("/templates/");
		Properties velocityProperties = new Properties();
		velocityProperties.put("file.resource.loader.cache", "false");
		velocityProperties.put("file.resource.loader.modificationCheckInterval", "0");
		velocityProperties.put("resource.manager.defaultcache.size", "0");
		velocityProperties.put("directive.set.null.allowed", "true");
		config.setVelocityProperties(velocityProperties);
		velocityProperties.put("output.encoding", "UTF-8");
		velocityProperties.put("input.encoding", "UTF-8");
		config.setVelocityProperties(velocityProperties);
		return config;
	}

	@Bean
	public VelocityViewResolver viewResolver() {
		VelocityViewResolver viewResolver = new VelocityViewResolver();
		viewResolver.setCache(false);
		viewResolver.setCacheUnresolved(false);
		viewResolver.setPrefix("/templates/");
		viewResolver.setSuffix(".html");
		viewResolver.setViewClass(VelocityView.class);
		viewResolver.setContentType("text/html;charset=UTF-8");
//		viewResolver.setToolboxConfigLocation("/WEB-INF/classes/config/toolbox.xml");
		return viewResolver;
	}

//	@Bean
//	public VelocityLayoutViewResolver viewResolver(){
//		VelocityLayoutViewResolver viewResolver= new VelocityLayoutViewResolver();
//		viewResolver.setPrefix("/templates/");
//		viewResolver.setSuffix(".htm");
//		viewResolver.setViewClass(VelocityLayoutView.class);
//		viewResolver.setLayoutUrl("/templates/layout/layout.vm");
//		viewResolver.setContentType("text/html;charset=UTF-8");
//		viewResolver.setToolboxConfigLocation("/WEB-INF/classes/config/toolbox.xml");
//		return viewResolver;
//	}
```

vm

所有的vm文件存放到src/main/resources的templates目录下。