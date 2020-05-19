# spring boot web

## 1. 创建准备

**注意：不要使用war创建项目，就使用普通的jar创建项目。**

### 1.1 pom.xml

```xml
		<!-- spring boot web -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
```

## 2. 工程目录结构

### 2.1 src/main/java

例如：sgw项目

cn.dongyuit.sgw(域名+应用简称)

cn.dongyuit.sgw.manager(域名+应用简称+模块)

cn.dongyuit.sgw.manager.web.controller(控制器包)

cn.dongyuit.sgw.manager.web.form(表单包)

cn.dongyuit.sgw.manager.logic(业务逻辑包)

cn.dongyuit.sgw.manager.domain(域对象包)

cn.dongyuit.sgw.manager.repository(持久化对象包)

cn.dongyuit.sgw.properties(属性配置包)

cn.dongyuit.sgw.web.MvcConfigurer(mvc配置类)

cn.dongyuit.sgw.web.ControllerExceptionAdvice(全局异常处理器)

cn.dongyuit.sgw.BeanConfiguration.java(bean声明和配置类)

cn.dongyuit.sgw.SgwManagerApplication(应用启动类)

#### 2.2.1 Controller(控制器)

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

###### 请求参数

如果有@PathVariable或者@RequestParam绑定，则参数类型不要使用基本类型，应该使用包装类型，例如：不要使用int类型的请求参数，应该使用Integer类型请求参数，否则在请求参数绑定**类型错误**的时候抛出的异常不同，基本类型(例如int)绑定类型异常抛出的是IllegalAgumentException，而包装类型(Integer)绑定类型异常错误抛出的是MethodArgumentTypeMismatchException，因为IllegalAgumentException异常太抽象了，java程序任何位置都可以抛出IllegalAgumentException异常，异常处理程序无法准确的失败抛出异常的原因，而MethodArgumentTypeMismatchException明确是web请求参数绑定类型匹配异常，处理处理程序可以准确的处理。**例如，参数应该是Integer Id，不应该是int id**。

###### @DateTimeFormat

日期类型请求参数绑定

```
@RequestParam @DateTimeFormat(pattern="yyyy-MM-dd HH:mm:ss") Date birthDay
```



```java
	@GetMapping(value = "/z1/service/demo/get", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
	public DemoData getDemoData(Integer id, @RequestParam(required = false) String name,
			@RequestParam(required = false) Date birthDay, @RequestParam(required = false) boolean sex,
			@RequestParam(required = false) BigDecimal wage) {
		DemoData demoData = new DemoData();
		demoData.setId(id);
		demoData.setName(name);
		demoData.setBirthDay(birthDay);
		demoData.setSex(sex);
		demoData.setWage(wage);
		return demoData;
	}
```



##### @PostMapping方法

**表单提交方法**：以XxxSubmit命令（例如：loginSubmit，提交登录操作），方法返回值是一个字符串(视图名)，参数：@Validated LoginForm loginForm页面表单项(请求参数)绑定的对象，@Validated源注释在绑定后会对参数进行有效性验证（具体下面介绍），BindingResult errors绑定失败错误对象，Model（用于存放model数据，页面显示使用），HttpSession session会话对象，可有可无根据实际业务情况定。

一般情况下：

第1步要执行errors.hasErrors()，先判断表单项绑定到form和form验证是否错误。

try{

第2步执行业务方法调用;    # 业务方法内业务异常会抛出BusinessException异常

第3步存放页面要显示的数据到model对象;

第4步返回视图名。

}catch(BusinessException businessException) {

   BusinessExceptionUtils.addErrors(businessException,errors); # 发送业务级异常,则把业务errorCode添加到Errors对象;

   return 当前表单页视图名;

}

例子，如下：

```java
@Controller
public class LoginController {

	@Autowired
	private SecurityLogic securityLogic;
    
	@PostMapping("/login.action")
	public String loginSubmit(@Validated LoginForm loginForm, BindingResult errors, Model model) {
		if (errors.hasErrors()) {
			return "/login";
		}
		try {
			User user = this.securityLogic.login(loginForm.getLoginName(), loginForm.getPassword());
			model.addAttribute("user", user);
			this.session.create();
			this.session.setAttribute("user", user);
			return "redirect:/index.action";
		} catch (BusinessException businessException) {
			BusinessExceptionUtils.addErrors(businessException, errors);
			return "/login";
		}

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

使用HttpSession作为controller方法的参数，你可以在controller方法内部操作"会话"。

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
        try{
	        User user = this.securityLogic.login(loginForm.getLoginName(), loginForm.getPassword());            
            model.addAttribute("user", user);
            session.setAttribute("user", user);  #　操作会话
            return "redirect:/index";            
        }catch(BusinessException businessException){
            BusinessExceptionUtils.addErrors(businessException,errors);
            return "login";
        }

	}

}
```

##### 拦截器(Interceptor)

编写一个拦截器，继承HandlerInterceptorAdapter，并且声明为Spring bean(@Component)，例如：

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

添加到mvc配置中，例如，下面的SecurityInterceptor。

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

##### 过滤器(filter)

###### FilterRegistrationBean方式

如果你的过滤器需要灵活的配置，例如：你写的过滤器是一个组件，供其他项目调研，则使用FilterRegistrationBean方式。

```java
@Configuration
@ConditionalOnProperty(value = "z1.web.filter.httpRequestDebug.enabled", havingValue = "true")
public class BeanConfiguration {

	@Value("${zw.web.filter.httpRequestDebug.name:httpRequestDebugFilter}")
	private String name;

	@Value("${zw.web.filter.httpRequestDebug.order:1}")
	private int order;

	@Value("${z1.web.filter.httpRequestDebug.urlPatterns:*.jsp,*.action,*.do}")
	private String[] urlPatterns;

	@Bean
	public FilterRegistrationBean<HttpRequestDebugFilter> filterRegistrationBean() {
		FilterRegistrationBean<HttpRequestDebugFilter> bean = new FilterRegistrationBean<>(
				new HttpRequestDebugFilter());
		bean.setName(this.name);
		bean.setOrder(this.order);
		bean.addUrlPatterns(this.urlPatterns);
		return bean;
	}

}

```

```java
public class HttpRequestDebugFilter extends OncePerRequestFilter {
	
	private static final Logger logger = LoggerFactory.getLogger(HttpRequestDebugFilter.class);

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		String debug = request.getParameter("debug");
		if("true".equalsIgnoreCase(debug)) {
			System.out.println(request);
			System.out.println(response);
		}
		filterChain.doFilter(request, response);
	}

}
```

###### @WebFilter方式

如果你写的过滤器，是为本项目使用，建议基于@WebFilter方式创建。

```java
@WebFilter(filterName = "test", urlPatterns = "/success/*")
public class UrlFilter implements Filter {
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
 
        System.out.println("----------------------->过滤器被创建");
    }
 
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
 
        HttpServletRequest req = (HttpServletRequest) servletRequest;
        String requestURI = req.getRequestURI();
        System.out.println("--------------------->过滤器：请求地址"+requestURI);
        if(!requestURI.contains("info")){
            servletRequest.getRequestDispatcher("/failed").forward(servletRequest, servletResponse);
        }else{
            filterChain.doFilter(servletRequest, servletResponse);
        }
    }
 
    @Override
    public void destroy() {
 
        System.out.println("----------------------->过滤器被销毁");
    }
}

```

需要加入@ServletComponentScan源注释

```java
@SpringBootApplication
@ServletComponentScan
public class Application {
	
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

```

##### RequestContextHolder

spring web 会自动注册RequestContextFilter的spring bean，其会把请求HttpServletRequest和HttpServletResponse对象存放到线程变量中。

你可以在请求流程经过的任何地方，调用如下代码，获取请求和响应对象

```java
((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();

((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getResponse();
```





#### 2.2.2 Form(表单)

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
	/**
    * Domain AppInfo -> ModifyAppForm
    *
    **/
	public ModifyAppForm(AppInfo appInfo) {
		this.id = appInfo.getId();
		this.appkey = appInfo.getAppkey();
		this.name = appInfo.getName();
		this.dayReqNumLimit = appInfo.getDayReqNumLimit();
		this.valid = appInfo.getValid();
	}
	/**
	* ModifyAppForm -> Domain AppInfo
	*
	**/
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
	/**
	* 表单的复杂验证
	*
	**/
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
		ModifyAppForm modifyAppForm = new ModifyAppForm(appInfo);  # 根据domain AppInfo创建form ModifyAppForm对象
		model.addAttribute("modifyForm", modifyAppForm);
		return "app/modify";
	}
```

##### toDomain()

参见上面的ModifyAppForm的toAppInfo()方法，可以把当前ModifyAppForm表单对象转换到AppInfo对象。AppInfo对象是一个Domain对象，是业务方法appInfoLogic.update的参数。例如：

注意：modifyAppForm.toAppInfo()操作。

```java
	@PostMapping("/app/modify.action")
	public String modifySubmit(@ModelAttribute("modifyForm") @Validated ModifyAppForm modifyAppForm,BindingResult errors, Model model) {
		if (errors.hasErrors() || modifyAppForm.validate(errors)) {
			return "/app/modify";
		}
		try {
            # 下面的modifyAppForm.toAppInfo()对象，根据form ModifyAppForm对象创建Domain AppInfo对象
			this.appInfoLogic.update(modifyAppForm.toAppInfo(), modifyAppForm.isChangeSecret());
			return "/app/modify_success";
		} catch (BusinessException businessException) {
			BusinessExceptionUtils.addErrors(businessException, errors);
			return "/app/modify";
		}
	}
```

##### validate(Errors errors)

对javax.validation声明式验证的补充操作，javax.validation和org.hibernate.validator只能对表单属性进行基本的验证，在进行复杂验证的时候，则力不从心，因此我们为form对象定义一个validate(Errors errors)方法对表单进行补充验证，javax.validation和org.hibernate.validator不能满足的地方，使用这个方法来验证，例如：参见上面的ModifyAppForm的validate(Errors erros)方法，其完成了只有changeSecret在为true的情况下，才需要验证secret项。

```java
	@PostMapping("/app/modify.action")
	public String modifySubmit(@ModelAttribute("modifyForm") @Validated ModifyAppForm modifyAppForm,BindingResult errors, Model model) {
        # 下面的modifyAppForm.validate(errors)实现了表单的复杂验证，并把错误信息写入到Spring Errors对象中。
		if (errors.hasErrors() || modifyAppForm.validate(errors)) {
			return "/app/modify";
		}
		try {
			this.appInfoLogic.update(modifyAppForm.toAppInfo(), modifyAppForm.isChangeSecret());
			return "/app/modify_success";
		} catch (BusinessException businessException) {
			BusinessExceptionUtils.addErrors(businessException, errors);
			return "/app/modify";
		}
	}
```

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

验证字符串不为空白，等同于!StringUtils.hasText(string)验证，**注意：只能验证字符串类型，否则报错**。

No validator could be found for constraint 'javax.validation.constraints.Size' validating type 'java.lang.Integer'

Integer类型的属性，使用@NotEmpty或者NotBlank来限制了，这是不对的，应该使用@NotNull

```
@NotEmpty(message = "不能为空")
```

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

#### 2.2.3 Logic(业务类)

几个原则：

1.表单form不应作为业务方法参数，应该使用domain对象作为参数，减少对mvc的耦合性。

2.业务方法内验证数据出错或者业务逻辑异常的时候(非系统异常)，应抛出BusinessException异常，并且尽量使用BusinessAssert(业务断言)来处理，代码规整而且好理解。Controller的方法会使用try catch来拦截logic方法抛出BusinessException异常并且添加到Errors对象中，再返回给表单页，具体见“@PostMapping方法章节”。

例如：

```java
@Service
@Transactional
public class SecurityLogic {
	/**　计算摘要的盐　*/
	private String salt = "xxxxx";

	@Autowired
	private DataDigest dataDigest;

	@Autowired
	private UserRepository userRepository;

	@Autowired
	private LogLogic logLogic;

	public User login(String loginName, String password) {
		User user = this.userRepository.findByLoginName(loginName);
		BusinessAssert.notNull(user, "loginName.notExist", "用户名不存在.");
		BusinessAssert.isTrue(user.isValid(), "loginName.invalid", "无效的用户.");
		String inputPasswordDigst = this.dataDigest.digestString((loginName + password + this.salt).getBytes());
		BusinessAssert.equals(user.getPassword(), inputPasswordDigst, "password.inputError", "密码错误.");
		this.logLogic.addLog(user, LogInfo.TYPE.LOGIN, "login", "loginName=" + loginName, String.valueOf(user.getId()));
		return user;
	}

}
```

##### 事务操作

参见：spring-boot respository的事务章节。

#### 2.2.4 Domain

一个模块内的domain放到同一个domain包下，Domain类的属性都应该是私有(private)并且通过GetSet方法来访问，并且要实现equals、hashcode和toString()方法。equals和hashcode方法应该使用自然键属性(唯一属性,但不是主键)，如果没有的可以使用主键属性，但要注意，在new一个domain对象并且没save的时候主键是null的。

例如：

```java
@Entity
@Table(name = "APP_INFO")
public class AppInfo {
	/** 编码 */
	@Id
	@GeneratedValue(generator = "gen_increment")
	@GenericGenerator(name = "gen_increment", strategy = "increment")
	@Column
	private Long id;
	/** app键值 */
	@Column
	private String appkey;
	/** 秘钥 */
	@Column
	private String secret;
	/** 名称 */
	@Column
	private String name;
	/** 日请求流量限制 */
	@Column(name = "day_req_num_limit")
	private Integer dayReqNumLimit;
	/** 有效性 */
	@Column
	private Boolean valid;
	/** 创建时间 */
	@Column(name = "created_time")
	private Date createdTime;
	/** 修改时间 */
	@Column(name = "updated_time")
	private Date updatedTime;

	public boolean isValidApp() {
		return this.valid != null && this.valid;
	}

	@Override
	public String toString() {
		return "AppInfo [id=" + id + ", appkey=" + appkey + ", secret=" + secret + ", name=" + name	+ ", dayReqNumLimit=" + dayReqNumLimit + ", valid=" + valid + ", createdTime=" + createdTime + ", updatedTime=" + updatedTime + "]";
	}

	@Override
	public int hashCode() {
		final int prime = 31;
		int result = 1;
		result = prime * result + ((id == null) ? 0 : id.hashCode());
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		AppInfo other = (AppInfo) obj;
		if (id == null) {
			if (other.id != null)
				return false;
		} else if (!id.equals(other.id))
			return false;
		return true;
	}

	public AppInfo copy() {
		AppInfo appInfo = new AppInfo();
		appInfo.id = this.id;
		appInfo.appkey = this.appkey;
		appInfo.secret = this.secret;
		appInfo.name = this.name;
		appInfo.dayReqNumLimit = dayReqNumLimit;
		appInfo.valid = this.valid;
		appInfo.createdTime = this.createdTime;
		appInfo.updatedTime = this.updatedTime;
		return appInfo;
	}

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getAppkey() {
		return appkey;
	}

	public void setAppkey(String appkey) {
		this.appkey = appkey;
	}

	public String getSecret() {
		return secret;
	}

	public void setSecret(String secret) {
		this.secret = secret;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Integer getDayReqNumLimit() {
		return dayReqNumLimit;
	}

	public void setDayReqNumLimit(Integer dayReqNumLimit) {
		this.dayReqNumLimit = dayReqNumLimit;
	}

	public Boolean getValid() {
		return valid;
	}

	public void setValid(Boolean valid) {
		this.valid = valid;
	}

	public Date getCreatedTime() {
		return createdTime;
	}

	public void setCreatedTime(Date createdTime) {
		this.createdTime = createdTime;
	}

	public Date getUpdatedTime() {
		return updatedTime;
	}

	public void setUpdatedTime(Date updatedTime) {
		this.updatedTime = updatedTime;
	}

}
```

#### 2.2.5 repository(数据访问类)

参见 “spring boot repository 文档”。

#### 2.2.6 properties(属性配置包)

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

#### 2.2.7 MvcConfigurer

对项目的Spring mvc框架使用的组件(bean)进行配置，例如：加入了一个拦截器

```java
@Configuration
public class MvcConfigurer implements WebMvcConfigurer {

	@Autowired
	private SecurityInterceptor securityInterceptor;

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		InterceptorRegistration userRegistration = registry.addInterceptor(this.securityInterceptor);
		userRegistration.addPathPatterns("/**/*.action");
	}
	
}

```

##### velocityResolver(velocity视图)

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

#### 2.2.8 ControllerExceptionAdvice

Controller(MVC)的全局异常处理器，对指定的异常进行处理，例如：

这是一个异常兜底拦截，对Throwable异常进行界面友好的可视化处理,并且记录日志。这里类以及集成到了Z1框架中，你只需要在你的项目中@Bean这个类，并且创建/error/exception页面就可以了。

```java
package z1.exception.spring.web.controller;

@ControllerAdvice(annotations = Controller.class)
public class ControllerExceptionAdvice {
	
	private static final Logger logger = LoggerFactory.getLogger(ControllerExceptionAdvice.class);

	@ExceptionHandler(Throwable.class)
	public String throwableHandle(Throwable tx, Model model) {
		logger.error("系统异常",tx);
		model.addAttribute("exception", tx);
		return "/error/exception";
	}

}
```

#### 2.2.9 BeanConfiguration

整个项目的Bean配置类，内部都是@Bean声明的方法，用于创建项目的使用spring bean，例如：RestTemplate、RedisTemplate等，例如：

```java
@Configuration
public class BeanConfiguration {

	@Bean
	public DataDigest dataDigest() {
		return new SHA256();
	}

	@Bean
	public Encryptor encryptor() {
		return new AES("xxxx");
	}

	@Bean
	public Session session() {
		return new HttpServletSession();
	}

}
```

#### 2.2.10 XxxYyyyyApplication

spring boot 应用启动类，例如：SgwManagerApplication

```java
@SpringBootApplication
@EnableJpaRepositories
@EnableTransactionManagement
@EnableViewVelocity
public class SgwManagerApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SgwManagerApplication.class, args);
	}
}
```



```java
@SpringBootApplication  # spring boot应用启动源注释
```

```
@EnableJpaRepositories  # 开启jpa
```

```
@EnableTransactionManagement # 开启事务
```

```
@EnableViewVelocity # 使用velocity,自定义，参加"2.2.7 MvcConfigurer的velcity视图配置"
```



### 2.2 src/main/resources

src/main/resource

src/main/resource/static (存放存静态文件，例如：js、css、images等)

src/main/resource/templates(存放视图文件，例如：xxx.html，扩展名上是html文件，其内部是html标签+视图脚本)   

src/main/resource/templates/vm_lib(如果是velocity环境,则存放velocity函数文件)

application.yml 配置文件（具体见，spring boot properties文档）

application-dev.yml 开发环境配置文件

application-test.yml 测试环境配置文件

application-proc.yml 生产环境配置文件



### @ControllerAdvice

对上一步的执行结果进行通知(拦截)处理。

#### @ControllerAdvice属性(条件)

@ControllerAdvice源注释提供了多个属性，如果使用(声明)了这些属性，就声明了某些**条件限制**，例如：@ControllerAdvice(annotations = { Z1ServiceRestController.class })限制了只对@Z1ServiceRestController声明的类执行进行通知(拦截)处理。如果没有声明任何属性，则对所有的执行结果进行通知(拦截)处理。

#### @ControllerAdvice的annotations属性

@ControllerAdvice的annotations属性值如果声明了多个，那么多个annotation是或(or)的关系，不是与(and)的关系，只要有一个annotation匹配上，ControllerAdvice就会运行。如果要使用and关系，则需要特殊处理，例如下面的Z1ServiceRestController源注释(@Interface)java类，其声明了@RestController，形成@RestController+Z1Service的and语义。

```java
@ControllerAdvice(annotations = { Z1ServiceRestController.class })
public class ServiceExceptionAdvice {
	@ExceptionHandler(Throwable.class)
	@ResponseBody
	public SystemErrorReturnValue throwableHandle(Throwable tx) {
		return this.serviceExceptionHandler.throwableHandle(tx);
	}
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RestController
public @interface Z1ServiceRestController {
}
```

#### @RestControllerAdvice

@ControllerAdvice+@ResponseBody的合体，意义同@RestController一样，是@Controller+@ResponseBody的合体。

#### @ResponseBody

如果你使用的是ControllerAdvice，并且要使用Rest方式处理异常返回值，则应该在异常处理方法上声明@ResponseBody，否则ControllerAdvice将使用默认的mvc方式输出异常信息到页面。你可以使用@RestControllerAdvice声明类，这样就无须@ReponseBody声明了。

```java
@ControllerAdvice(annotations = { Z1ServiceRestController.class })
public class ServiceExceptionAdvice {
	@ExceptionHandler(Throwable.class)
	@ResponseBody
	public SystemErrorReturnValue throwableHandle(Throwable tx) {
		return this.serviceExceptionHandler.throwableHandle(tx);
	}
}
```

#### ResponseBodyAdvice

对上一步的执行结果,响应输出前进行通知(拦截)处理，这里的上一步可以是RestController方法执行结果，也可以是上一个ControllerAdvice的执行结果(例如：异常通知(拦截)处理)，例如，下面例子annotations声明的annotations = { Z1ServiceRestController.class, ControllerAdvice.class}两个类型。

```java
@ControllerAdvice(annotations = { Z1ServiceRestController.class, ControllerAdvice.class })
public class ServiceReturnValueResponseBodyAdvice implements ResponseBodyAdvice<Object> {

	@Autowired
	private RequestIdContext requestIdContext;

	@Override
	public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
		return true;
	}

	@Override
	public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request,
			ServerHttpResponse response) {

		String requestId = this.requestIdContext.get();
		if (body instanceof ErrorReturnValue) {
			// 服务错误返回值,直接返回
			((ErrorReturnValue) body).setRequestId(requestId);
			return body;
		} else {
			// 包装返回业务对象到服务成功返回值
			SuccessReturnValue successReturnValue = new SuccessReturnValue();
			successReturnValue.setRequestId(requestId);
			successReturnValue.setData(body);
			return successReturnValue;
		}

	}

}
```

### RequestBodyAdvice

对rest请求对象，进行拦截处理，例如：只获取服务请求对象中data对象(业务对象)，传递给restcontroller方法的请求参数。

```

```





## 2.RestController







### Z1Service

基于z1框架依托Spring的RestController提供了服务化(Z1Service)，实现了服务化的服务端，而且对程序无侵入性，客户端你可以基于RestTemplate调用。

#### @Z1Service + @RestController

在@RestController声明的类上加入@Z1Service源注释，标记当前这个RestController是z1框架的服务。

```

```

#### @GetMapping(获取数据服务)

```java

```

#### @PostMapping(提交数据服务)



#### 服务返回值处理



#### 服务异常统一处理

#### Form(表单)

完全同"1.2 "

#### Logic(业务类)



##### 序列化和反序列化

你可以通过调试WebMvcConfigurationSupport类的getMessageConverters()方法,来查看当前使用的转换器(HttpMessageConverter)。排在前面的转换器优先会执行。

org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport

```java
	protected final List<HttpMessageConverter<?>> getMessageConverters() {
		if (this.messageConverters == null) {
			this.messageConverters = new ArrayList<>();
			configureMessageConverters(this.messageConverters);  # 重写方法1
			if (this.messageConverters.isEmpty()) { #注意:如果你重写类方法1,则下面的添加默认转换器不会执行了。
				addDefaultHttpMessageConverters(this.messageConverters);
			}
			extendMessageConverters(this.messageConverters);  # 重写访问2
		}
		return this.messageConverters;
	}
```

正常情况下，你额可以通过重写两个方法来实现转换器注入，重写方法：

如果重写这个方法，则添加默认转换器的addDefaultHttpMessageConverters方法就不会执行了。

```java
	protected void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
	}

```

如果重写这个方法，则可以添加转换器，但添加在了默认的转换器的后面，会被最后选择执行。

```java
	protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
	}
```

##### 输出一个图片或文件

1.源注释@RequestMapping配置consumes属性，指定输出文件的类型(content-type)，例如：这里配置为了输出PNG类型。

2.返回值为Resource类型，然后你实例化一个Resource的实现类，例如：这里创建了FileSystemResource对象，spring会使用ResourceHttpMessageConverter转换器来输出。

```java
	@RequestMapping(value = "/testResource", consumes = MediaType.ALL_VALUE,produces = MediaType.IMAGE_PNG_VALUE)
	public Resource testResource(@RequestParam int a) {
		return new FileSystemResource("C:\\Users\\Administrator\\Desktop\\重要\\T_SERVICE_LOG.png");
	}
```



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