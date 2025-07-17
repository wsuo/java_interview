# Redis实现原理与常用数据结构

## 1. Redis基本概念

Redis（Remote Dictionary Server）是一个开源的内存数据结构存储系统，可用作数据库、缓存和消息代理。

### 1.1 Redis特点
- **内存存储**：数据存储在内存中，读写速度极快
- **持久化**：支持RDB和AOF两种持久化方式
- **数据结构丰富**：支持多种数据类型
- **单线程模型**：避免锁竞争，保证操作原子性
- **支持集群**：主从复制、哨兵模式、Cluster模式

## 2. Redis数据结构详解

### 2.1 String（字符串）

**实现原理：**
- 底层使用SDS（Simple Dynamic String）实现
- 预分配空间，减少内存重新分配次数
- 二进制安全，可存储任何数据

**常用命令：**
```redis
SET key value
GET key
INCR key
DECR key
MGET key1 key2
MSET key1 value1 key2 value2
```

**应用场景：**
- 缓存用户信息
- 计数器（点赞数、访问量）
- 分布式锁

**Java示例：**
```java
@Autowired
private RedisTemplate<String, String> redisTemplate;

// 设置用户信息
public void setUserInfo(String userId, String userInfo) {
    redisTemplate.opsForValue().set("user:" + userId, userInfo, Duration.ofHours(1));
}

// 计数器
public Long incrementCounter(String key) {
    return redisTemplate.opsForValue().increment(key);
}
```

### 2.2 Hash（哈希）

**实现原理：**
- 小哈希表使用ziplist（压缩列表）
- 大哈希表使用hashtable
- 自动转换阈值：hash-max-ziplist-entries=512, hash-max-ziplist-value=64

**常用命令：**
```redis
HSET key field value
HGET key field
HMGET key field1 field2
HGETALL key
HDEL key field
```

**应用场景：**
- 存储对象属性
- 购物车
- 用户配置信息

**Java示例：**
```java
// 存储用户对象
public void setUser(String userId, User user) {
    Map<String, String> userMap = new HashMap<>();
    userMap.put("name", user.getName());
    userMap.put("email", user.getEmail());
    userMap.put("age", String.valueOf(user.getAge()));
    
    redisTemplate.opsForHash().putAll("user:" + userId, userMap);
}

// 购物车操作
public void addToCart(String userId, String productId, Integer quantity) {
    redisTemplate.opsForHash().put("cart:" + userId, productId, quantity.toString());
}
```

### 2.3 List（列表）

**实现原理：**
- 3.2版本前：ziplist + linkedlist
- 3.2版本后：quicklist（ziplist的双向链表）
- 支持两端插入和删除

**常用命令：**
```redis
LPUSH key value
RPUSH key value
LPOP key
RPOP key
LRANGE key start stop
LLEN key
```

**应用场景：**
- 消息队列
- 最新动态列表
- 栈和队列实现

**Java示例：**
```java
// 消息队列
public void sendMessage(String queue, String message) {
    redisTemplate.opsForList().leftPush(queue, message);
}

public String receiveMessage(String queue) {
    return redisTemplate.opsForList().rightPop(queue);
}

// 最新文章列表
public void addArticle(String articleId) {
    redisTemplate.opsForList().leftPush("latest:articles", articleId);
    redisTemplate.opsForList().trim("latest:articles", 0, 99); // 保持100条
}
```

### 2.4 Set（集合）

**实现原理：**
- 整数集合：intset（元素都是整数且数量较少）
- 哈希表：hashtable

**常用命令：**
```redis
SADD key member
SREM key member
SISMEMBER key member
SMEMBERS key
SINTER key1 key2
SUNION key1 key2
```

**应用场景：**
- 标签系统
- 好友关系
- 去重计数

**Java示例：**
```java
// 用户标签
public void addUserTag(String userId, String tag) {
    redisTemplate.opsForSet().add("user:tags:" + userId, tag);
}

// 共同好友
public Set<String> getCommonFriends(String userId1, String userId2) {
    return redisTemplate.opsForSet().intersect("friends:" + userId1, "friends:" + userId2);
}

// 文章点赞用户
public void likeArticle(String articleId, String userId) {
    redisTemplate.opsForSet().add("article:likes:" + articleId, userId);
}
```

### 2.5 ZSet（有序集合）

**实现原理：**
- 跳跃表（skiplist）+ 哈希表
- 跳跃表保证有序性
- 哈希表保证O(1)查找

**常用命令：**
```redis
ZADD key score member
ZREM key member
ZRANGE key start stop
ZRANGEBYSCORE key min max
ZRANK key member
```

**应用场景：**
- 排行榜
- 延时队列
- 范围查询

**Java示例：**
```java
// 游戏排行榜
public void updateScore(String userId, Integer score) {
    redisTemplate.opsForZSet().add("leaderboard", userId, score);
}

public Set<String> getTopPlayers(int count) {
    return redisTemplate.opsForZSet().reverseRange("leaderboard", 0, count - 1);
}

// 延时队列
public void addDelayedTask(String taskId, long delaySeconds) {
    long executeTime = System.currentTimeMillis() + delaySeconds * 1000;
    redisTemplate.opsForZSet().add("delayed:tasks", taskId, executeTime);
}
```

## 3. Redis高级特性

### 3.1 持久化机制

**RDB（Redis Database）：**
- 快照持久化，保存某个时间点的数据
- 适合备份和灾难恢复
- 文件紧凑，恢复速度快

**AOF（Append Only File）：**
- 记录每个写操作
- 数据安全性更高
- 文件大，恢复速度较慢

**混合持久化（4.0+）：**
```redis
# 配置示例
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
```

### 3.2 发布订阅模式

**基本使用：**
```java
// 发布者
@Service
public class MessagePublisher {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public void publishMessage(String channel, String message) {
        redisTemplate.convertAndSend(channel, message);
    }
}

// 订阅者
@Component
public class MessageSubscriber {
    
    @EventListener
    public void handleMessage(String message) {
        System.out.println("Received: " + message);
    }
}
```

### 3.3 Lua脚本

**原子性操作：**
```java
// 分布式锁实现
public boolean tryLock(String key, String value, long expireTime) {
    String script = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "  return redis.call('del', KEYS[1]) " +
        "else " +
        "  return 0 " +
        "end";
    
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    
    Long result = redisTemplate.execute(redisScript, 
                                      Collections.singletonList(key), 
                                      value);
    return result != null && result == 1L;
}
```

## 4. 面试常见问题

### Q1: Redis为什么这么快？
1. **内存存储**：避免磁盘IO
2. **单线程模型**：避免线程切换开销
3. **非阻塞IO**：epoll模型
4. **优化的数据结构**：针对不同场景优化

### Q2: Redis单线程如何处理并发？
- 使用IO多路复用（epoll）
- 事件驱动模型
- 6.0版本引入多线程处理网络IO

### Q3: Redis内存淘汰策略？
```redis
# 8种淘汰策略
noeviction       # 不淘汰，内存满时报错
allkeys-lru      # 所有key中LRU淘汰
allkeys-lfu      # 所有key中LFU淘汰
allkeys-random   # 所有key中随机淘汰
volatile-lru     # 过期key中LRU淘汰
volatile-lfu     # 过期key中LFU淘汰
volatile-random  # 过期key中随机淘汰
volatile-ttl     # 过期key中TTL最小的淘汰
```

### Q4: Redis事务特性？
- **原子性**：要么全部执行，要么全部不执行
- **一致性**：事务执行前后数据一致
- **隔离性**：事务执行不受其他事务影响
- **不支持持久性**：取决于持久化策略

```java
// 事务示例
public void transferMoney(String fromAccount, String toAccount, Integer amount) {
    redisTemplate.execute(new SessionCallback<Object>() {
        @Override
        public Object execute(RedisOperations operations) throws DataAccessException {
            operations.watch("account:" + fromAccount);
            
            String balance = (String) operations.opsForValue().get("account:" + fromAccount);
            if (Integer.parseInt(balance) >= amount) {
                operations.multi();
                operations.opsForValue().decrement("account:" + fromAccount, amount);
                operations.opsForValue().increment("account:" + toAccount, amount);
                return operations.exec();
            }
            
            operations.unwatch();
            return null;
        }
    });
}
```

## 5. 生产环境实践

### 5.1 性能优化
```java
// 连接池配置
@Configuration
public class RedisConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        GenericObjectPoolConfig poolConfig = new GenericObjectPoolConfig();
        poolConfig.setMaxTotal(200);
        poolConfig.setMaxIdle(50);
        poolConfig.setMinIdle(10);
        poolConfig.setTestOnBorrow(true);
        
        LettuceClientConfiguration clientConfig = LettuceClientConfiguration.builder()
            .poolConfig(poolConfig)
            .commandTimeout(Duration.ofSeconds(2))
            .build();
        
        RedisStandaloneConfiguration serverConfig = new RedisStandaloneConfiguration();
        serverConfig.setHostName("localhost");
        serverConfig.setPort(6379);
        
        return new LettuceConnectionFactory(serverConfig, clientConfig);
    }
}
```

### 5.2 缓存策略
```java
// 缓存穿透解决方案
public User getUserById(String userId) {
    String key = "user:" + userId;
    String userJson = redisTemplate.opsForValue().get(key);
    
    if (userJson != null) {
        if ("null".equals(userJson)) {
            return null; // 缓存空值
        }
        return JSON.parseObject(userJson, User.class);
    }
    
    User user = userService.findById(userId);
    if (user != null) {
        redisTemplate.opsForValue().set(key, JSON.toJSONString(user), Duration.ofMinutes(30));
    } else {
        redisTemplate.opsForValue().set(key, "null", Duration.ofMinutes(5)); // 缓存空值
    }
    
    return user;
}

// 缓存击穿解决方案（分布式锁）
public User getUserByIdWithLock(String userId) {
    String key = "user:" + userId;
    String lockKey = "lock:user:" + userId;
    
    String userJson = redisTemplate.opsForValue().get(key);
    if (userJson != null) {
        return JSON.parseObject(userJson, User.class);
    }
    
    // 获取分布式锁
    String lockValue = UUID.randomUUID().toString();
    Boolean lockAcquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10));
    
    if (lockAcquired) {
        try {
            // 再次检查缓存
            userJson = redisTemplate.opsForValue().get(key);
            if (userJson != null) {
                return JSON.parseObject(userJson, User.class);
            }
            
            // 从数据库加载
            User user = userService.findById(userId);
            if (user != null) {
                redisTemplate.opsForValue().set(key, JSON.toJSONString(user), Duration.ofMinutes(30));
            }
            return user;
        } finally {
            // 释放锁
            releaseLock(lockKey, lockValue);
        }
    } else {
        // 等待并重试
        try {
            Thread.sleep(100);
            return getUserByIdWithLock(userId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
}
```

## 6. 集群与高可用

### 6.1 主从复制
```redis
# 从节点配置
replicaof 192.168.1.100 6379
replica-read-only yes
```

### 6.2 哨兵模式
```java
// Sentinel配置
@Configuration
public class SentinelConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisSentinelConfiguration sentinelConfig = 
            new RedisSentinelConfiguration("mymaster", 
                Set.of("192.168.1.101:26379", "192.168.1.102:26379", "192.168.1.103:26379"));
        
        return new LettuceConnectionFactory(sentinelConfig);
    }
}
```

### 6.3 Cluster模式
```java
// Cluster配置
@Configuration
public class ClusterConfig {
    
    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration();
        clusterConfig.clusterNode("192.168.1.101", 7000);
        clusterConfig.clusterNode("192.168.1.102", 7001);
        clusterConfig.clusterNode("192.168.1.103", 7002);
        
        return new LettuceConnectionFactory(clusterConfig);
    }
}
```