# Spring boot actuator

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

