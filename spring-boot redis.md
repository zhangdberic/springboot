# spring boot redis

## 1.安装和配置

### spring boot redis有两种客户端

1.非阻塞的lettuce客户端，线程安全的非阻塞redis客户端，一般情况无须配置连接池，但如果你的代码中有长时间占用redis的代码，例如：lua、redis事务等，最好还是配置上连接池。

2.jedis阻塞客户端，阻塞的非线程安全的redis客户端，这个必须配置连接池了。

### 1.1 pom.xml

#### lettuce客户端(建议)

spring boot redis默认使用了lettuce客户端，默认是单连接情况，如果你需要配置连接池则需要加入commons-pool2。

```xml
		<!-- spring boot redis -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>	
		<!-- lettuce pool use -->
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-pool2</artifactId>
		</dependency>		
```

#### jedis客户端

你需要显示的去掉lettuce-core并加入jedis客户端，然后加入commons-pool2支持连接池。

```xml
		<!-- spring boot redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <!-- 排除lettuce包，使用jedis代替-->
            <exclusions>
                <exclusion>
                    <groupId>io.lettuce</groupId>
                    <artifactId>lettuce-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
		<!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
		<dependency>
		    <groupId>redis.clients</groupId>
		    <artifactId>jedis</artifactId>
		</dependency>
        		
		<!-- lettuce pool use -->
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-pool2</artifactId>
		</dependency>	
```

### 1.2 application.yml

#### lettuce客户端(建议)

lettuce单连接情况，无须配置pool，这在一般的小型的管理系统足够了。

```yaml
spring:
  # redis    
  redis:
    host: 39.105.202.78 
    port: 6400
    timeout: 3000
    password: xxxxx
```

lettuce连接池，lettect单连接就已经很强了，因此你的连接池不能配置的太大，过多的连接无用。

```yaml
spring:
  # redis    
  redis:
    host: 39.105.202.78 
    port: 6400
    timeout: 3000
    password: Redis-401
    lettuce:
      pool:
        minIdle: 4
        maxIdle: 4
        maxWait: 1000
        maxActive: 20
        timeBetweenEvictionRuns: 30000
      shutdownTimeout: 100
```

lettuce连接池基于apache commons-pool2实现，具体的参数可以参见"apache commons-pool2.md"文档。

minIdle： 最小空闲连接数，默认0；

maxIdle： 最大空闲连接数，默认8；

maxWait： 获取连接等待时间(ms)，默认-1无限等待，这个必须调整；

maxActive：最大连接数上限；

timeBetweenEvictionRuns： 清理(evict)线程执行的时间间隔毫秒(spring boot默认为null)，必须设置这个值否则无法开启evict线程，开启这个线程很重要。注意：lettuce的连接池evict相关的属性很多，但spring boot data redis只能设置timeBetweenEvictionRuns属性，其它属性都是commons-pool2的默认值，例如：testWhileIdle=false、numTestsPerEvictionRun=3，查看"apache commons-pool2.md"文档。

shutdownTimeout：关闭lettuct客户端的时候,阻塞100毫秒,给所有lettuce连接时间让其正常关闭,超出了则强制退出。这个一般情况无须设置。

#### jedis客户端

jedis必须基于连接池实现，否则性能根本无法保证。连接配置对比lettuce没有什么变化，就是连接池的lettuce改为jedis。

```yaml
spring:
  # redis sgw   
  redis:
    host: 192.168.5.54
    port: 6379
    timeout: 1000
    password: 12345678
    jedis:
      pool:
        minIdle: 1
        maxIdle: 1
        maxWait: 1000
        maxActive: 20
        timeBetweenEvictionRuns: 30000
```

jedis连接池基于apache commons-pool2实现，具体的参数可以参见"apache commons-pool2.md"文档。

minIdle： 最小空闲连接数，默认0；

maxIdle： 最大空闲连接数，默认8；

maxWait： 获取连接等待时间(ms)，默认-1无限等待，这个必须调整；

maxActive：最大连接数上限；

timeBetweenEvictionRuns： 清理(evict)线程执行的时间间隔毫秒(spring boot默认为null)，必须设置这个值否则无法开启evict线程，开启这个线程很重要。

不同于lettuce连接池，jedis基于JedisPoolConfig来配置连接池，JedisPoolConfig默认已经设置了如下属性：

查看"apache commons-pool2.md"文档，了解下面的属性。

```java
  public JedisPoolConfig() {
    // defaults to make your life with connection pool easier :)
    setTestWhileIdle(true);
    setMinEvictableIdleTimeMillis(60000);
    setTimeBetweenEvictionRunsMillis(30000);
    setNumTestsPerEvictionRun(-1);
  }
```



### 1.3 RedisTemplate

```java
	@Autowired
	private RedisTemplate<String, String> redisTemplate;
```

## 2.RedisTemplate方法使用

### hmset

```java
		Map<String, String> hashFields = new LinkedHashMap<String, String>();
		hashFields.put("secret", appInfo.getSecret());
		hashFields.put("dayReqNumLimit", String.valueOf(appInfo.getDayReqNumLimit()));
		this.redisTemplate.opsForHash().putAll("app_" + appInfo.getAppkey(), hashFields);
```



### 管道(pipelined)

注意返回值：管道内调用redis的命令的返回值都是null，你不应在RedisCallback回调函数内依赖于redis命令的返回值，例如：管道内connection.incr(xxx)命令返回null，pipelined会把其内部每个redis命令的返回值存放到List<Object>，你可以通过List<Object> results来获取每个redis命令的返回值（按照redis命令调用顺序）。

```java
			List<Object> results = this.redisTemplate.executePipelined(new RedisCallback<Void>() {
				@Override
				public Void doInRedis(RedisConnection connection) throws DataAccessException {
					connection.incr(appDayReqCountKey.getBytes());
					connection.incr(serDayReqCountKey.getBytes());
					connection.incr(appSerDayReqCountKey.getBytes());
					return null;
				}
			});
			requestNumCounter.setAppDayCount((Long) results.get(0));
			requestNumCounter.setServiceDayCount((Long) results.get(1));
			requestNumCounter.setAppServiceDayCount((Long) results.get(2));
```

### 脚本调用(lua script)

DefaultRedisScript是spring提供的，其存放redis lua脚本字符串，计算sha1，定义返回值类型。

注意：redisTemplate.execute的keys和args都不能为null，如果null修改为new Object[0]。

```java
		DefaultRedisScript<Long> defaultRedisScript = new DefaultRedisScript<>();
		defaultRedisScript.setScriptSource(
				new ResourceScriptSource(new ClassPathResource("z1/tool/rate_limiter/RateLimiter.lua")));
		defaultRedisScript.setResultType(Long.class);
		List<String> keys = Arrays
				.asList(new String[] { key, String.valueOf(permits),             String.valueOf(System.currentTimeMillis()),						String.valueOf(RedisLuaRateLimiter.this.rateLimiterConfig.getMaxPermits()),						String.valueOf(RedisLuaRateLimiter.this.rateLimiterConfig.getRateTimeunit()),			String.valueOf(RedisLuaRateLimiter.this.rateLimiterConfig.getRateNum()) });
		Long retValue = this.redisTemplate.execute(defaultRedisScript, keys, new Object[0]);
		return retValue == 1;
	
```

## 3.redis lua script

### redis lua 判断redis的get和hget返回值否为空

get或hget返回空的时候，返回的不是nil，而是boolean类型。

为空判断

```lua
if(curr_num == nil or (type(curr_num) == 'boolean' and not curr_num)) then
end
```

不为空判断

```lua
if (type(curr_num) ~= 'boolean' and curr_num ~= false and curr_num ~= nil) then
end
```

### redis.call("common",xx,xx)返回值类型

首先明确一点，在redis系统中值的存储的永远都是字符串，但其操作**命令的返回值**类型可以是多种，对照如下：

| redis返回值类型 |              Lua数据类型               |
| :-------------: | :------------------------------------: |
|    整数回复     |                数字类型                |
|   字符串回复    |               字符串类型               |
| 多行字符串回复  |          table类型(数组形式)           |
|    状态回复     | table类型(只有一个ok字段存储状态信息)  |
|    错误回复     | table类型(只有一个err字段存储错误信息) |

你可以通过redis-cli命令，做实验，操作后通过查看返回值，来识别redis命令返回值，例如：

```
172.17.28.215:6400> get key1
"1"
```

因为是双引号，明显就是返回值类型为字符串。

```
172.17.28.215:6400> incr key1
(integer) 2
```

直观的告诉你就是integer类型。

#### redis命令返回值和lua类型对照例子

| redis.call("common",xxx)命令返回值                           | lua类型                                               |
| ------------------------------------------------------------ | ----------------------------------------------------- |
| redis.call("GET","key1")                                     | string                                                |
| redis.call("INCR","key1")                                    | number                                                |
| local fields = redis.pcall("HMGET", "key1", "field1", "field2")    <br/>local field1= fields[1]   <br/>local field2= fields[2] | fields为table<br/> field1为string<br/> field2为string |

### 调试

可以在需要位置加入：redis.call("SET","debug1",xxxx)，然后使用redis-cli，来查看debug1的值。

### 限流脚本

```lua
-- 基于令牌桶限流的lua脚本
-- 代码参照了google java Guava 的RateLimiter限流类

-- 返回码
-- -1 表示取令牌失败，也就是桶里没有令牌
-- 1 表示取令牌成功

-- 参数列表
local key = KEYS[1] -- 限流的KEY
local permits = tonumber(KEYS[2]) -- 请求许可(令牌)数量
local currt_time_millis = tonumber(KEYS[3]) -- 当前时间(毫秒数)
local max_permits = tonumber(KEYS[4]) -- 令牌桶的最大令牌数(限制)
local rate_timeunit = tonumber(KEYS[5]) -- 速率时间单位(秒数)
local rate_num = tonumber(KEYS[6]) -- 速率个数，每个速率时间单位产生的令牌数量，例如:10 


-- 程序代码
local rate_limit_info = redis.pcall("HMGET", key, "last_time_millis", "curr_permits")    
local last_time_millis = rate_limit_info[1]    -- 上一次添加令牌时间(毫秒数)
local curr_permits = rate_limit_info[2]   -- 当前剩余令牌数 

local available_permits = max_permits    -- 可用令牌数,初始化为最大令牌数

-- redis lua判断是否为空比较奇葩,需要先判断是否为boolean类型,下面的if语句判断last_time_millis不为空情况
if (type(last_time_millis) ~= 'boolean' and last_time_millis ~= false and last_time_millis ~= nil) then
    -- 下面三行代码,模拟了按照固定的速率放令牌到令牌桶的过程(每秒放多少个令牌到令牌桶,如果令牌桶已经满了就丢弃),
    -- 第三行代码,比较特殊其有两层含义:1.如果令牌桶已经满了就丢弃.2.计算当前可用的令牌数.
    last_time_millis = tonumber(last_time_millis)
    curr_permits = tonumber(curr_permits)
    
    local reverse_permits = math.floor(((currt_time_millis - last_time_millis) / (rate_timeunit*1000)) * rate_num)  -- 距离上次访问时间，需要产生的令牌数(距离长产生的令牌多,距离短产生的令牌少,在同一个rate_num内为零)       
    local expect_curr_permits = reverse_permits + curr_permits; -- 这段时间产生的令牌数 + 上次剩余的令牌数
    available_permits = math.min(expect_curr_permits, max_permits);  -- 重新计算可用令牌数，不能超过最大令牌数    

    if (reverse_permits > 0) then      --- 大于0表示,1个或多个rate_timeunit时间周期已结过去,需要重新更新最后访问时间为当前时间
        redis.pcall("HSET", key, "last_time_millis", currt_time_millis)
    end
else
    redis.pcall("HSET", key, "last_time_millis", currt_time_millis)   -- 初始化(第一次访问)最后访问时间为当前时间
end
    
local result = -1
if (available_permits - permits >= 0) then
    -- 可用令牌数大于要获取的令牌数
    result = 1
    redis.pcall("HSET", key, "curr_permits", available_permits - permits)    -- 设置剩余的可用令牌数
else
    redis.pcall("HSET", key, "curr_permits", available_permits)   -- 重置当前令牌数为可用令牌数,并返回-1,获取令牌(许可)失败 
end

local expire_time = rate_timeunit * 2 -- 过期时间，时间单位*2，每次请求都会刷新过期时间
redis.pcall("EXPIRE",key,expire_time)  -- 设置过期时间,防止撑爆redis

return result


```

### 请求数量限制脚本

```lua
-- 请求测试限制

-- 返回值
--  -1 表示失败，超出了请求数限制
--  大于0 表示成功，返回当前数量

-- 参数列表
local key = KEYS[1] -- redis的key
local delta = tonumber(KEYS[2]) -- incr的步长
local max_num = tonumber(KEYS[3]) -- 限制数量
local expire_time = tonumber(KEYS[4]) -- key过期时间(秒),-1则不设置过期时间

-- 程序代码
local curr_num = redis.call("GET", key)
if(curr_num == nil or (type(curr_num) == 'boolean' and not curr_num)) then
  curr_num = 0
else
  curr_num = tonumber(curr_num)
end

if(curr_num >= max_num) then
   return -1
else
   curr_num = redis.call("INCR", key)
   if (expire_time ~= -1) then
       if(curr_num == 1) then
          redis.pcall("EXPIRE",key,expire_time)
       end 
   end
   return curr_num
end
```

