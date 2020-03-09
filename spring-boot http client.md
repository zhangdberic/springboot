# spring-boot http client

# RestTemplate

### pom.xml

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

## 连接池实现

**好文章：**

https://blog.51cto.com/liukang/2090211

https://blog.csdn.net/zzzgd_666/article/details/88858181

https://www.cnblogs.com/myitnews/p/12195340.html

```java
@RestController
    public class HelloController {
        private final String TARGET_HOST = "http://localhost:8092";
        private RestTemplate restTemplate;

        public HelloController() {  // 1
            PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
            connectionManager.setDefaultMaxPerRoute(1000);
            connectionManager.setMaxTotal(1000);
            this.restTemplate = new RestTemplate(new HttpComponentsClientHttpRequestFactory(                    HttpClientBuilder.create().setConnectionManager(connectionManager).build()
            ));

        }

        @GetMapping("/hello/{latency}")
        public String hello(@PathVariable int latency) {
            return restTemplate.getForObject(TARGET_HOST + "/hello/" + latency, String.class);
        }
    }
```



