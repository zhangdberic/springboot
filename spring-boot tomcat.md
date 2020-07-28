# spring boot tomcat

## yaml

### 端口设置

```
server:
  port: 7070
```

### 字符集设置

```yaml
server:
  tomcat: 
    uri-encoding: UTF-8
spring: 
  http: 
    encoding: 
      charset: UTF-8
      enabled: true
      force: true

```

### 线程数设置

下面的设置都是默认值，正常情况下不需要调整。

```yaml
server:
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    accept-count: 100
    max-connections: 8192
```

### 访问日志设置

server.tomcat.accesslog.enabled默认(false)是关闭的，你需要显示的设置为true来开启；

默认已经开启了日志循环(rotate = true)；

directory：配置访问日志文件存放的目录；

maxDays：配置日志保留的天数，默认为-1保留以前所有的滚动日志文件；

buffered：写入缓冲区(默认值true)，如果为调试方便可以设置为false(日志写入马上可见)。

renameOnRotate：直到开始滚动了(日期切换)才修改日志文件名为滚动文件名，默认为false。设置为true的时候，当天的日志文件名为access_log.log，否则为access_log.2020-07-16.log。

```yaml
server:
  tomcat:
    accesslog:
      enabled: true
      encoding: UTF-8
      prefix: ${spring.application.name}_access_log
      directory: /home/sgw/logs
      maxDays: 180
      renameOnRotate: true
      pattern: "%a %{X-Forwarded-For}i %{yyyyMMddHHmmss}t %r %{Referer}i %{User-Agent}i %s %b %{Content-Type}o [%D ms]"
```

注意：开启访问日志会记录两个文件，一个是普通的日志访问文件(access_log文件名前缀)，另一个是actuator请求访问日志文件(management_access_log文件名前置)，例如：access_log.2020-07-16.log，management_access_log.2020-07-16.log。



#### pattern

建议设置如下：

```
%a %{X-Forwarded-For}i %t "%r" %s %b %T "%{Referer}i" "%{User-Agent}i"
```

pattern属性为日志内容格式，默认为**common**；

- **%a** - Remote IP address，远程ip地址，注意不一定是原始ip地址，中间可能经过nginx等的转发
- **%A** - Local IP address，本地ip
- **%b** - Bytes sent, excluding HTTP headers, or '-' if no bytes were sent
- **%B** - Bytes sent, excluding HTTP headers
- **%h** - Remote host name (or IP address if `enableLookups` for the connector is false)，远程主机名称(如果resolveHosts为false则展示IP)
- **%H** - Request protocol，请求协议
- **%l** - Remote logical username from identd (always returns '-')
- **%m** - Request method，请求方法（GET，POST）
- **%p** - Local port，接受请求的本地端口
- **%q** - Query string (prepended with a '?' if it exists, otherwise an empty string
- **%r** - First line of the request，HTTP请求的第一行（包括请求方法，请求的URI）
- **%s** - HTTP status code of the response，HTTP的响应代码，如：200,404
- **%S** - User session ID
- **%t** - Date and time, in Common Log Format format，日期和时间，Common Log Format格式
- **%u** - Remote user that was authenticated
- **%U** - Requested URL path
- **%v** - Local server name
- **%D** - Time taken to process the request, in millis，处理请求的时间，单位毫秒
- **%T** - Time taken to process the request, in seconds，处理请求的时间，单位秒
- **%I** - current Request thread name (can compare later with stacktraces)，当前请求的线程名，可以和打印的log对比查找问题

Access log 也支持将cookie、header、session或者其他在ServletRequest中的对象信息打印到日志中，其配置遵循Apache配置的格式（{xxx}指值的名称）：

- `%{xxx}i` for incoming headers，request header信息
- `%{xxx}o` for outgoing response headers，response header信息
- `%{xxx}c` for a specific cookie
- `%{xxx}r` xxx is an attribute in the ServletRequest
- `%{xxx}s` xxx is an attribute in the HttpSession
- `%{xxx}t` xxx is an enhanced SimpleDateFormat pattern (see Configuration Reference document for details on supported time patterns)

Access log内置了两个日志格式模板，可以直接指定pattern为模板名称，如：`server.tomcat.accesslog.pattern=common`：

- **common** - `%h %l %u %t "%r" %s %b`，依次为：远程主机名称，远程用户名，被认证的远程用户，日期和时间，请求的第一行，response code，发送的字节数
- **combined** - `%h %l %u %t "%r" %s %b "%{Referer}i" "%{User-Agent}i"`，依次为：远程主机名称，远程用户名，被认证的远程用户，日期和时间，请求的第一行，response code，发送的字节数，request header的Referer信息，request header的User-Agent信息。

除了内置的模板，我们常用的配置有：

- `%t %a "%r" %s (%D ms)`，日期和时间，请求来自的IP（不一定是原始IP），请求第一行，response code，响应时间（毫秒），样例：`[21/Mar/2017:00:06:40 +0800] 127.0.0.1 POST /bgc/syncJudgeResult HTTP/1.0 200 63`，这里请求来自IP就是经过本机的nginx转发的。
- `%t [%I] %{X-Forwarded-For}i %a %r %s (%D ms)`，日期和时间，线程名，**原始IP**，请求来自的IP（不一定是原始IP），请求第一行，response code，响应时间（毫秒），样例：`[21/Apr/2017:00:24:40 +0800][http-nio-7001-exec-4] 10.125.15.1 127.0.0.1 POST /bgc/syncJudgeResult HTTP/1.0 200 5`，这里的第一个IP是Nginx配置了`X-Forwarded-For`记录了原始IP。

这里简要介绍下上面用到的HTTP请求头`X-Forwarded-For`，它是一个 HTTP 扩展头部，用来表示 HTTP 请求端真实 IP，其格式为：`X-Forwarded-For: client, proxy1, proxy2`，其中的值通过一个`逗号+空格`把多个IP地址区分开，最左边（client）是最原始客户端的IP地址，代理服务器每成功收到一个请求，就把请求来源IP地址添加到右边。

## keepalive(连接保持)

tomcat默认已经开启了keepalived，默认的Keep-Alive: timeout=60，MaxKeepAliveRequests=100。要使keepalive有效，要求客户端也要支持keepalived，例如：apache ab默认不支持，要加入-k参数。

## TomcatServletWebServerFactory

tomcat提供了很多yaml配置来修改tomcat参数，但还有很多参数不能通过yaml来设置，那么就需要使用本类通过代码来修改。

例如：修改keepalive属性

```java
@Component
public class WebServerConfiguration implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        //使用对应工厂类提供给我们的接口定制化我们的tomcat connector
        ((TomcatServletWebServerFactory)factory).addConnectorCustomizers(new TomcatConnectorCustomizer() {
            @Override
            public void customize(Connector connector) {
                Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();

                //定制化keepAlivetimeout, 设置30 秒内没有请求则服务器自动断开keepalive链接
                protocol.setKeepAliveTimeout(30000);
                //当客户端发送超过10000个请求则自动断开keepalive链接。
                protocol.setMaxKeepAliveRequests(10000);
            }
        });
    }
}
```

