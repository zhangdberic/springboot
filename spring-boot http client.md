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

### 开启调试

```yaml
logging:
  level:
    root: INFO 
    org.apache.http: DEBUG 
```

### 构建RestTemplate的Bean

基于z1框架的z1.tool.httpclient.apache包内代码实现。

z1.tool.httpclient.apache.HttpClientProperties为属性配置类；你可以通过查看代码来查看支持的属性设置；

z1.tool.httpclient.apache.HttpClientFactory为httpClient的工厂类，其创建HttpClient对象，做为RestTemplate底层通讯工具。

例如：

```java
import z1.tool.httpclient.apache.HttpClientConfig;
import z1.tool.httpclient.apache.HttpClientFactory;

@Configuration
@ConfigurationProperties(prefix = "fss.thumb-image.resttemplate")
public class FssThumbImageRestTemplateConfiguration extends HttpClientProperties {
	
	@Bean
	public RestTemplate thumbImageRestTemplate() {
		HttpClient httpClient = new HttpClientFactory(this).build();
		RestTemplate restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
		return restTemplate;
	}
	
}
```

你可以通过yaml配置文件，来配置这个RestTemplate客户端，例如：

```yaml
fss:
  thumb-image:
    resttemplate:
      connectTimeout: 2000
      readTimeout: 10000
      
```

#### 连接池实现

参见“HttpClientProperties”文档，z1框架已经封装了实现，你只需要修改yaml配置就可以了，默认已经开启了连接池。

最好通过查看HttpClientProperties类代码，来深入了解连接池相关的配置属性。

例如：

```yaml
fss:
  thumb-image:
    resttemplate:
      maxConnTotal: 10
      maxConnPerRoute: 10
      keepAliveTime: 60
      keepAliveTargetHost:
        "www.163.com": 10
        "www.lnyg.net": 20
      
```

#### https支持

参见“HttpClientProperties”文档，z1框架已经封装了实现，你只需要修改yaml配置就可以了。

例如：

```yaml
fss:
  thumb-image:
    resttemplate:
      allowUntrustedCerts: false
```



### 请求URI或URL构建

Spring RestTemplate提供了URI uri和String URL两种来声明请求发送的位置。两种方式有区别，URI方式的请求Spring RestTemplate不再进行URI变量替换和uri encode参数编码，而String URL发送的请求Spring RestRemplate会进行URL变量替换和url encode参数编码。



**基于Restful URI参数变量的构建**

例如：

```java
this.restTemplate.getForObject("http://localhost/app/{appId}", App.class, 999l);
```

restTemplate的URI参数

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

### 构建application/json请求

```java
	private ResponseEntity<ResponseInfo> internalPostJson(String action, String json) {
		URL requestUrl = this.urlGenerator.generate(action, null);
		HttpHeaders headers = new HttpHeaders();
		headers.setContentType(MediaType.APPLICATION_JSON);
		headers.add("appkey", this.clientProperties.getAppkey());
		HttpEntity<String> requestEntity = new HttpEntity<>(json,headers);
		ResponseEntity<ResponseInfo> responseEntity = null;
		try {
			responseEntity = this.elicenseClientRestTemplate.exchange(requestUrl.toString(), HttpMethod.POST,requestEntity, ResponseInfo.class);
		} catch (Exception ex) {
			throw new RestTemplateExchangeException(requestUrl, requestEntity, responseEntity,
					"dzzz service client execute error.", ex);
		}
		if (!HttpStatus.OK.equals(responseEntity.getStatusCode())) {
			throw new RestTemplateExchangeException(requestUrl, requestEntity, responseEntity,
					"http response status code[" + responseEntity.getStatusCodeValue() + "] error.");
		}
		if (logger.isDebugEnabled()) {
			logger.debug("request url[{}], requestEntity[{}], responseEntity[{}].", requestUrl, requestEntity,
					responseEntity);
		}
		return responseEntity;
	}
	
```

### 指定响应内容格式

如果服务端是spring mvc rest构建的服务，则客户端可以使用，如下三种方式来指定响应内容格式：

1.请求URL扩展名：例如：请求原地址URL为/get，那么想获取json内容格式，请求URL为/get.json，获取xml内容格式，请求URL为/get.xml。

2.请求参数format：例如：请求原地址URL为/get，那么想获取json内容格式，请求URL为/get?format=json，获取xml内容格式，请求URL为/get?format=xml。

3.请求头Accept：例如：请求原地址URL为/get，那么想获取json内容格式，加入请求头Accept:application/json，获取xml内容格式，请求头Accept:application/xml。例如：RestTemplate默认Accept是根据RestTemplate定义MessageConverter来生成的，也就是我有什么消息解析器，我就使用Accept头，通知服务器。如果服务器端只支持json格式输出，那你就显示的设置Accept:application/json，如下：

```java
		HttpHeaders headers = new HttpHeaders();
		List<MediaType> accept = new ArrayList<>();
		accept.add(MediaType.APPLICATION_JSON);
		headers.setAccept(accept);
		HttpEntity<Void> requestEntity = new HttpEntity<>(headers);
```

详细介绍参见：spring-boot mvc.md文档。



### 发送请求

#### exchange请求

你可以添加请求头、请求参数，可以获取响应状态码、响应头、序列化后的响应对象。

```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
requestEntity = new HttpEntity<>(headers);
MultiValueMap<String, Object> params = new LinkedMultiValueMap<String, Object>();
params.add("name","黑哥");
params.add("password","你说呢");
HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<>(params, headers);
```

#### execute(RequestCallback,ResponseExtractor)请求

这是一个相对底层的方法，提供了底层的通讯协议，RequestCallback提供了请求的回调操作，你可以直接操作请求的输出流，ResponseExtractor提供了响应的回调操作，你可以直接操作响应的输入流。

```java
			this.restTemplate.execute(getFssFileUri, HttpMethod.GET, new RequestCallback() {

				@Override
				public void doWithRequest(ClientHttpRequest request) throws IOException {
					request.getBody().write("黑哥".getBytes());
				}
			}, new ResponseExtractor<String>() {

				@Override
				public String extractData(ClientHttpResponse response) throws IOException {
					InputStream inputStream = response.getBody();
					return IoUtils.copyToString(inputStream, Charset.forName("UTF-8"));
				}
			});
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

