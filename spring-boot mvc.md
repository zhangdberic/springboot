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

#### controller(控制器)

下面是一个典型的控制器类，其实现了登录控制：

**进入页面的方法**：以XxxForm命名（例如：loginForm，进入登录页面操作），方法返回值是一个字符串(视图名)，参数为Model（用于存放model数据，页面显示使用），URL映射@GetMapping源注释，URL后缀使用xxx.action。

**表单提交方法**：以XxxSubmit命令（例如：loginSubmit，提交登录操作），方法返回值是一个字符串(视图名)，参数：@Validated LoginForm loginForm页面表单项(请求参数)绑定的对象，@Validated源注释在绑定后会对参数进行有效性验证（具体下面介绍），BindingResult errors绑定失败错误对象，Model（用于存放model数据，页面显示使用），HttpSession session会话对象，可有可无根据实际业务情况定。

一般情况下，第1步要执行errors.hasErrors()，先判断表单项绑定到form和form验证是否错误。

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

#### 表单Form对象

form对象有一个原则，form对象不能和domain对象有任何关联，例如：不应继承与domain对象，也不能使用domain对象作为属性等，这样可以减少耦合性，让form对象和domain对象各司其职。

例如：登录表单Form对象

```java
public class LoginForm {

	@NotBlank(message="用户名不能为空")
	@Size(max=20,message="用户名不正确")
	private String loginName;

	@NotBlank(message="密码不能为空")
	@Size(min=6,max=20,message="密码不正确")
	private String password;

	public String getLoginName() {
		return loginName;
	}

	public void setLoginName(String loginName) {
		this.loginName = loginName;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

}
```



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