# spring-boot http client

## RestTemplate

### pom.xml

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

### 使用apache http client为底层通讯客户端

```xml
		<!-- apache http client -->
		<dependency>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpclient</artifactId>
		</dependency>
```

### 连接池实现

基于z1框架的z1.tool.httpclient.apache包内代码实现。

z1.tool.httpclient.apache.HttpClientProperties为属性配置类；

z1.tool.httpclient.apache.HttpClientFactory为httpClient的工厂类，其创建HttpClient对象，做为RestTemplate底层通讯工具。

如下，使用z1框架集成RestTemplate+apache http client连接池。

```java
import z1.tool.httpclient.apache.HttpClientConfig;
import z1.tool.httpclient.apache.HttpClientFactory;

@Configuration
@ConfigurationProperties(prefix = "fss.thumb-image.resttemplate")
public class FssThumbImageRestTemplateConfiguration extends HttpClientConfig {
	
	@Bean
	public RestTemplate thumbImageRestTemplate() {
		HttpClient httpClient = new HttpClientFactory(this).build();
		RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
		return restTemplate;
	}
	
}
```

### 请求URI构建

**基于Restful URI参数变量的构建**

例如：

```java
this.restTemplate.getForObject("http://localhost/app/{appId}", App.class, 999l);
```

**URI的Query部分参数构建**

restTemplate不提供URI的query部分参数构建，也就是默认不支持?arg1=xxx&args2=xxx这样的构建，需要自己实现。

spring-boot-service-client项目可以提供参照

```java
					MultiValueMap<String, String> queryParams = new LinkedMultiValueMap<String, String>();
                    queryParams.add("arg1",xxx);
                    queryParams.add("arg2",xxx);
					requestUri = UriComponentsBuilder.fromUri(requestUri).queryParams(queryParams).build().encode()
							.toUri();
```

### 请求头构建

```java
			HttpHeaders headers = new HttpHeaders();
			headers.setContentType("application/x-www-form-urlencoded");
			requestEntity = new HttpEntity<>(headers);
```

### 构建x-www-form-urlencoded表单对象

spring-boot-service-client项目可以提供参照

```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);  // 关键
requestEntity = new HttpEntity<>(headers);
MultiValueMap<String, Object> params = new LinkedMultiValueMap<String, Object>();
params.add("name","黑哥");
params.add("password","你说呢");
HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<>(params, headers);
```

### 构建multipart/form-data表单对象

spring-boot-service-client项目可以提供参照

```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.MULTIPART_FORM_DATA); // 关键
requestEntity = new HttpEntity<>(headers);
MultiValueMap<String, Object> params = new LinkedMultiValueMap<String, Object>();
params.add("name","黑哥");
params.add("password","你说呢");
// 关键，上传文件
FileSystemResource fileSystemResource1 = new FileSystemResource("C:\\T_SERVICE_LOG_DATA2.png");
params.add("file1",fileSystemResource1);        
HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<>(params, headers);
```

### 发送请求

#### exchange模式

spring-boot-service-client项目可以提供参照

```java
			ResponseEntity<User> responseEntity = this.serviceClientRestTemplate.exchange(requestUri, method,
					requestEntity, User.class);
```

### 请求和响应对象转换器(HttpMessageConverter)

spring-boot-service-client项目可以提供参照

RestTemplate默认创建的时候，会同步创建多个转换器，例如：自动配置了Jackson转换器（前提你的pom.xml中加了boot-starter-web）。

请求和响应内容到对象转换器，主要是通过请求和响应的Content-Type来选择的，例如：Content-Type=application/json，那么会使用MappingJackson2HttpMessageConverter转换器，注意：如果要使用jackson来支持xml，则需要在pom.xml中再加入：

```xml
		<dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-xml</artifactId>
		</dependency>
```

#### 自定义HttpMessageConverter

例如：我们实现了一个服务请求客户端，需要自定义响应内容的转换，把响应内容先临时保存，然后交个调用者来根据具体情况反序列化，代码如下：

```java
public class ResponseInputStreamHttpMessageConverter extends AbstractHttpMessageConverter<InputStream> {

	public ResponseInputStreamHttpMessageConverter() {
		super(MediaType.ALL);
	}

	@Override
	protected boolean supports(Class<?> clazz) {
		return true;
	}

	@Override
	public boolean canRead(Class<?> clazz, @Nullable MediaType mediaType) {
		return true;
	}

	@Override
	protected InputStream readInternal(Class<? extends InputStream> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {
		InputStream inputStream = inputMessage.getBody();
		OfflineInputStream offlineInputStream = new OfflineInputStream();
		offlineInputStream.offline(inputStream);
		return offlineInputStream;
	}

	@Override
	public boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType) {
		return false;
	}

	@Override
	protected void writeInternal(InputStream t, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {
		throw new java.lang.UnsupportedOperationException(
				"only read response content, should not be used as request content.");
	}

}
```

这里的核心代码是OfflineInputStream，其实现了响应输入流的离线处理，具体代码见：

z1.util.stream.OfflineInputStream

如上转换器需要添加到RestTemplate中，如下：

```java
	@Bean
	public RestTemplate serviceClientRestTemplate(ServiceClientProperties serviceClientProperties) {
		HttpClient httpClient = new HttpClientFactory(serviceClientProperties.getHttpClientConfig()).build();
		RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
		List<HttpMessageConverter<?>> messageConverters = new ArrayList<>();
		messageConverters.add(new ResponseInputStreamHttpMessageConverter());
		messageConverters.add(new AllEncompassingFormHttpMessageConverter());
		restTemplate.setMessageConverters(messageConverters);
		return restTemplate;
	}

```

AllEncompassingFormHttpMessageConverter实现了表单的输入转换器(请求内容转换器),ResponseInputStreamHttpMessageConverter实现了响应内容转换器。RestTemplate调用示例代码如下：

```java
			ResponseEntity<InputStream> responseEntity = this.serviceClientRestTemplate.exchange(requestUri, method,
					requestEntity, InputStream.class);
			InputStream responseInputStream = responseEntity.getBody();
```

