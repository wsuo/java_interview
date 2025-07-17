# Spring框架全面解析

## 1. Spring框架概述

### 1.1 Spring核心模块

```
Spring Framework
├── Core Container
│   ├── spring-core
│   ├── spring-beans
│   ├── spring-context
│   └── spring-expression
├── Data Access/Integration
│   ├── spring-jdbc
│   ├── spring-tx
│   ├── spring-orm
│   └── spring-jms
├── Web
│   ├── spring-web
│   ├── spring-webmvc
│   └── spring-websocket
├── AOP
│   ├── spring-aop
│   └── spring-aspects
└── Test
    └── spring-test
```

### 1.2 Spring Boot vs Spring Framework

| 特性 | Spring Framework | Spring Boot |
|------|------------------|-------------|
| 配置 | XML/Java配置 | 自动配置 |
| 依赖管理 | 手动添加 | Starter依赖 |
| 服务器 | 外部服务器 | 内嵌服务器 |
| 监控 | 需要集成 | Actuator |
| 开发速度 | 慢 | 快 |

## 2. IOC容器详解

### 2.1 IOC核心概念

**控制反转（Inversion of Control）：**
- 对象的创建和管理交给框架
- 降低代码耦合度
- 提高可测试性

**依赖注入（Dependency Injection）：**
- 构造器注入
- Setter注入
- 字段注入

### 2.2 IOC容器实现

```java
// BeanFactory - 基础容器
public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    boolean containsBean(String name);
}

// ApplicationContext - 高级容器
public interface ApplicationContext extends BeanFactory {
    String getApplicationName();
    Resource[] getResources(String locationPattern) throws IOException;
    void publishEvent(ApplicationEvent event);
    MessageSource getMessageSource();
}
```

### 2.3 Bean的生命周期

```java
@Component
public class BeanLifecycleDemo implements BeanNameAware, BeanFactoryAware, 
        ApplicationContextAware, InitializingBean, DisposableBean {
    
    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;
    
    // 1. 构造器
    public BeanLifecycleDemo() {
        System.out.println("1. Bean构造器执行");
    }
    
    // 2. 设置Bean属性
    @Value("${demo.property:default}")
    public void setProperty(String property) {
        System.out.println("2. 设置Bean属性: " + property);
    }
    
    // 3. Aware接口回调
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("3. BeanNameAware.setBeanName(): " + name);
    }
    
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
        System.out.println("4. BeanFactoryAware.setBeanFactory()");
    }
    
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        System.out.println("5. ApplicationContextAware.setApplicationContext()");
    }
    
    // 4. BeanPostProcessor前置处理
    // 由容器调用，不需要实现
    
    // 5. InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("6. InitializingBean.afterPropertiesSet()");
    }
    
    // 6. @PostConstruct
    @PostConstruct
    public void init() {
        System.out.println("7. @PostConstruct方法执行");
    }
    
    // 7. 自定义初始化方法
    @Bean(initMethod = "customInit")
    public void customInit() {
        System.out.println("8. 自定义初始化方法");
    }
    
    // 8. BeanPostProcessor后置处理
    // 由容器调用，不需要实现
    
    // 9. Bean使用阶段
    public void doSomething() {
        System.out.println("9. Bean使用阶段");
    }
    
    // 10. @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("10. @PreDestroy方法执行");
    }
    
    // 11. DisposableBean.destroy()
    @Override
    public void destroy() throws Exception {
        System.out.println("11. DisposableBean.destroy()");
    }
    
    // 12. 自定义销毁方法
    @Bean(destroyMethod = "customDestroy")
    public void customDestroy() {
        System.out.println("12. 自定义销毁方法");
    }
}
```

### 2.4 依赖注入实现

```java
// 构造器注入（推荐）
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}

// Setter注入
@Service
public class OrderService {
    private UserService userService;
    private PaymentService paymentService;
    
    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
    
    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// 字段注入（不推荐）
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private CategoryService categoryService;
}
```

## 3. AOP面向切面编程

### 3.1 AOP核心概念

- **切面（Aspect）**：横切关注点的模块化
- **连接点（Join Point）**：程序执行的某个特定位置
- **切点（Pointcut）**：连接点的集合
- **通知（Advice）**：切面在特定连接点执行的代码
- **织入（Weaving）**：把切面应用到目标对象的过程

### 3.2 AOP实现方式

```java
// 1. 基于注解的AOP
@Aspect
@Component
public class LoggingAspect {
    
    // 定义切点
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // 前置通知
    @Before("serviceLayer()")
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("Before: " + joinPoint.getSignature().getName());
    }
    
    // 后置通知
    @After("serviceLayer()")
    public void afterAdvice(JoinPoint joinPoint) {
        System.out.println("After: " + joinPoint.getSignature().getName());
    }
    
    // 返回通知
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void afterReturningAdvice(JoinPoint joinPoint, Object result) {
        System.out.println("AfterReturning: " + result);
    }
    
    // 异常通知
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "exception")
    public void afterThrowingAdvice(JoinPoint joinPoint, Exception exception) {
        System.out.println("AfterThrowing: " + exception.getMessage());
    }
    
    // 环绕通知
    @Around("serviceLayer()")
    public Object aroundAdvice(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("Around: Before");
        Object result = joinPoint.proceed();
        System.out.println("Around: After");
        return result;
    }
}
```

### 3.3 自定义注解实现

```java
// 自定义注解
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MethodTimer {
    String value() default "";
    boolean logParams() default false;
    boolean logResult() default false;
}

// 切面实现
@Aspect
@Component
public class MethodTimerAspect {
    
    @Around("@annotation(methodTimer)")
    public Object timeMethod(ProceedingJoinPoint joinPoint, MethodTimer methodTimer) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        // 记录参数
        if (methodTimer.logParams()) {
            Object[] args = joinPoint.getArgs();
            System.out.println("Method params: " + Arrays.toString(args));
        }
        
        try {
            // 执行方法
            Object result = joinPoint.proceed();
            
            // 记录结果
            if (methodTimer.logResult()) {
                System.out.println("Method result: " + result);
            }
            
            return result;
        } finally {
            long endTime = System.currentTimeMillis();
            System.out.println("Method " + joinPoint.getSignature().getName() + 
                             " executed in " + (endTime - startTime) + "ms");
        }
    }
}

// 使用示例
@Service
public class UserService {
    
    @MethodTimer(value = "getUserById", logParams = true, logResult = true)
    public User getUserById(Long id) {
        // 业务逻辑
        return userRepository.findById(id);
    }
}
```

## 4. Spring常用注解

### 4.1 核心注解

```java
// 组件扫描
@ComponentScan("com.example")
@Configuration
public class AppConfig {
    
    @Bean
    @Primary // 主要候选Bean
    @Qualifier("userService") // 指定Bean名称
    public UserService userService() {
        return new UserServiceImpl();
    }
    
    @Bean
    @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
    public FeatureService featureService() {
        return new FeatureService();
    }
}

// 条件注解
@Component
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(name = "redis.enabled", havingValue = "true")
public class RedisService {
    // Redis服务实现
}

// 配置属性
@ConfigurationProperties(prefix = "app.database")
@Component
public class DatabaseConfig {
    private String url;
    private String username;
    private String password;
    private int maxConnections;
    
    // getters and setters
}
```

### 4.2 Web相关注解

```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
        User user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(@PathVariable Long id, 
                                         @Valid @RequestBody UpdateUserRequest request) {
        User user = userService.update(id, request);
        return ResponseEntity.ok(user);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping
    public ResponseEntity<List<User>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String keyword) {
        
        PageRequest pageRequest = PageRequest.of(page, size);
        List<User> users = userService.findAll(keyword, pageRequest);
        return ResponseEntity.ok(users);
    }
}
```

### 4.3 数据访问注解

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    Optional<User> findByEmail(String email);
    
    @Query(value = "SELECT * FROM users WHERE status = :status", nativeQuery = true)
    List<User> findByStatus(@Param("status") String status);
    
    @Modifying
    @Query("UPDATE User u SET u.lastLoginTime = :time WHERE u.id = :id")
    int updateLastLoginTime(@Param("id") Long id, @Param("time") LocalDateTime time);
}

@Service
@Transactional
public class UserService {
    
    @Transactional(readOnly = true)
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }
    
    @Transactional(rollbackFor = Exception.class)
    public User create(CreateUserRequest request) {
        // 验证邮箱唯一性
        if (userRepository.findByEmail(request.getEmail()).isPresent()) {
            throw new DuplicateEmailException("Email already exists");
        }
        
        User user = new User();
        user.setEmail(request.getEmail());
        user.setName(request.getName());
        user.setCreatedTime(LocalDateTime.now());
        
        return userRepository.save(user);
    }
}
```

## 5. SpringCloud组件

### 5.1 服务注册与发现

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

// Eureka Client
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// 服务调用
@RestController
public class OrderController {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/order/{orderId}")
    public Order getOrder(@PathVariable String orderId) {
        // 获取用户服务实例
        List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
        ServiceInstance instance = instances.get(0);
        
        // 调用用户服务
        String userServiceUrl = instance.getUri() + "/users/" + orderId;
        User user = restTemplate.getForObject(userServiceUrl, User.class);
        
        return new Order(orderId, user);
    }
}
```

### 5.2 服务熔断与降级

```java
// Hystrix熔断器
@Component
public class UserServiceFallback {
    
    @HystrixCommand(fallbackMethod = "getUserFallback")
    public User getUser(Long userId) {
        // 远程调用用户服务
        return restTemplate.getForObject("/users/" + userId, User.class);
    }
    
    public User getUserFallback(Long userId) {
        // 降级处理
        User fallbackUser = new User();
        fallbackUser.setId(userId);
        fallbackUser.setName("Unknown User");
        return fallbackUser;
    }
}

// Resilience4j断路器
@Component
public class UserService {
    
    @CircuitBreaker(name = "user-service", fallbackMethod = "getUserFallback")
    public User getUser(Long userId) {
        return restTemplate.getForObject("/users/" + userId, User.class);
    }
    
    public User getUserFallback(Long userId, Exception ex) {
        return new User(userId, "Fallback User");
    }
}
```

### 5.3 API网关

```java
// Zuul网关
@SpringBootApplication
@EnableZuulProxy
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}

// 自定义过滤器
@Component
public class AuthenticationFilter extends ZuulFilter {
    
    @Override
    public String filterType() {
        return "pre";
    }
    
    @Override
    public int filterOrder() {
        return 1;
    }
    
    @Override
    public boolean shouldFilter() {
        return true;
    }
    
    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        
        String token = request.getHeader("Authorization");
        
        if (token == null || !isValidToken(token)) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("Authentication required");
            return null;
        }
        
        return null;
    }
}

// Spring Cloud Gateway
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r.path("/users/**")
                .uri("lb://user-service"))
            .route("order-service", r -> r.path("/orders/**")
                .uri("lb://order-service"))
            .build();
    }
}
```

### 5.4 配置中心

```java
// Config Server
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}

// Config Client
@RestController
@RefreshScope
public class ConfigController {
    
    @Value("${app.message:default message}")
    private String message;
    
    @GetMapping("/config")
    public String getConfig() {
        return message;
    }
}
```

## 6. 面试常见问题

### Q1: Spring IOC的实现原理？

**答案：**
1. **Bean定义读取**：解析XML或注解配置
2. **Bean实例化**：通过反射创建Bean实例
3. **依赖注入**：设置Bean的属性和依赖
4. **初始化**：调用初始化方法
5. **使用**：Bean可以被应用程序使用
6. **销毁**：容器关闭时销毁Bean

### Q2: Spring AOP的实现原理？

**答案：**
Spring AOP基于代理模式实现：
- **JDK动态代理**：针对接口的代理
- **CGLIB代理**：针对类的代理
- **织入时机**：运行时织入

### Q3: Spring循环依赖如何解决？

**答案：**
Spring使用三级缓存解决循环依赖：
```java
// 一级缓存：完整的Bean实例
Map<String, Object> singletonObjects = new ConcurrentHashMap<>();

// 二级缓存：早期Bean实例
Map<String, Object> earlySingletonObjects = new HashMap<>();

// 三级缓存：Bean工厂
Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>();
```

### Q4: @Autowired和@Resource的区别？

| 特性 | @Autowired | @Resource |
|------|------------|-----------|
| 来源 | Spring | JSR-250 |
| 注入方式 | 按类型 | 按名称 |
| 配合注解 | @Qualifier | @Named |
| 支持位置 | 构造器、字段、方法 | 字段、方法 |

### Q5: Spring事务的传播行为？

```java
public enum Propagation {
    REQUIRED,      // 如果当前有事务，加入；如果没有，新建
    SUPPORTS,      // 如果当前有事务，加入；如果没有，非事务执行
    MANDATORY,     // 如果当前有事务，加入；如果没有，抛异常
    REQUIRES_NEW,  // 总是新建事务
    NOT_SUPPORTED, // 总是非事务执行
    NEVER,         // 总是非事务执行，如果有事务抛异常
    NESTED         // 嵌套事务
}
```

### Q6: SpringBoot自动配置原理？

**答案：**
1. **@EnableAutoConfiguration**：启用自动配置
2. **spring.factories**：配置自动配置类
3. **@Conditional**：条件判断
4. **@ConfigurationProperties**：属性绑定

```java
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        return template;
    }
}
```