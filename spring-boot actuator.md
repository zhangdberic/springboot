# Spring boot actuator

## 接收请求的地址和端口

```yaml
management:
  server:
    address: 127.0.0.1
    port: 18080  
```

## 设置端点(功能)

include: "*"，设置所有的功能都开发，你可以设置：include: "env,beans,health"

```yaml
management:
  endpoints:
    web:
      exposure:
        include:
        - "*"
```



## 安全shutdown

### application.yml

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
      base-path: /cH8DxFOH8Wme4XHl
  server:
    address: 127.0.0.1
    port: 18080      
```

### 本机发送post请求关键

```
/usr/bin/curl -X POST http://127.0.0.1:18080/cH8DxFOH8Wme4XHl/shutdown
```

## 健康检查

### 关闭redis健康检查

```yaml
management:
  health:
    redis:
      enabled: false
```

