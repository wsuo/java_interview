# JWT实现详解

## 1. JWT基本概念

JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在网络应用环境间安全地传输信息。

### 1.1 JWT结构
JWT由三部分组成，用点号分隔：
```
Header.Payload.Signature
```

- **Header**：包含算法和token类型
- **Payload**：包含声明（claims）
- **Signature**：用于验证token的完整性

### 1.2 JWT工作原理
1. 用户登录时，服务器验证身份后生成JWT
2. 客户端在后续请求中携带JWT
3. 服务器验证JWT的有效性和完整性
4. 如果验证通过，处理请求

## 2. Spring Boot中的JWT实现

### 2.1 依赖配置
```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
</dependency>
```

### 2.2 JWT工具类
```java
@Component
public class JwtUtils {
    
    private final String secret = "mySecretKey";
    private final long expiration = 86400000; // 24小时
    
    // 生成Token
    public String generateToken(String username) {
        return Jwts.builder()
            .setSubject(username)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS512, secret)
            .compact();
    }
    
    // 验证Token
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(secret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    // 获取用户名
    public String getUsernameFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody()
            .getSubject();
    }
}
```

### 2.3 JWT过滤器
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                  HttpServletResponse response, 
                                  FilterChain filterChain) throws ServletException, IOException {
        
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            
            if (jwtUtils.validateToken(token)) {
                String username = jwtUtils.getUsernameFromToken(token);
                
                // 设置认证信息
                UsernamePasswordAuthenticationToken authentication = 
                    new UsernamePasswordAuthenticationToken(username, null, new ArrayList<>());
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }
        
        filterChain.doFilter(request, response);
    }
}
```

## 3. 面试常见问题

### Q1: JWT的优缺点？
**优点：**
- 无状态，服务端不需要存储
- 可扩展，支持跨域
- 自包含，包含用户信息

**缺点：**
- 无法撤销（除非维护黑名单）
- 信息泄露风险
- 续签复杂

### Q2: JWT vs Session？
| 特性 | JWT | Session |
|------|-----|---------|
| 存储位置 | 客户端 | 服务端 |
| 扩展性 | 好 | 差 |
| 撤销能力 | 困难 | 容易 |
| 安全性 | 中等 | 高 |

### Q3: 如何解决JWT续签问题？
1. **双Token机制**：Access Token + Refresh Token
2. **滑动过期**：每次请求重新生成Token
3. **定时刷新**：客户端定时请求新Token

## 4. 安全最佳实践

### 4.1 密钥管理
- 使用强密钥（至少256位）
- 定期轮换密钥
- 环境变量存储密钥

### 4.2 Token安全
- 设置合理的过期时间
- 使用HTTPS传输
- 避免在URL中传输Token
- 客户端安全存储（HttpOnly Cookie）

### 4.3 验证增强
```java
// 添加更多验证
public boolean validateToken(String token, String expectedAudience) {
    try {
        Claims claims = Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
        
        // 验证audience
        return expectedAudience.equals(claims.getAudience());
    } catch (Exception e) {
        return false;
    }
}
```

## 5. 实际应用场景

### 5.1 微服务认证
在微服务架构中，JWT作为统一的认证方式：
```java
@RestController
public class AuthController {
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        // 验证用户凭证
        if (authService.validate(request.getUsername(), request.getPassword())) {
            String token = jwtUtils.generateToken(request.getUsername());
            return ResponseEntity.ok(new AuthResponse(token));
        }
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }
}
```

### 5.2 API网关集成
```java
@Component
public class GatewayJwtFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());
        
        if (token != null && jwtUtils.validateToken(token)) {
            // 添加用户信息到请求头
            ServerHttpRequest request = exchange.getRequest().mutate()
                .header("X-User-Id", jwtUtils.getUsernameFromToken(token))
                .build();
            
            return chain.filter(exchange.mutate().request(request).build());
        }
        
        return unauthorized(exchange);
    }
}
```

## 6. 故障排查

### 6.1 常见问题
1. **Token过期**：检查系统时间和过期时间设置
2. **签名验证失败**：确认密钥一致性
3. **格式错误**：检查Token格式和编码

### 6.2 调试技巧
```java
// 添加详细日志
public boolean validateToken(String token) {
    try {
        Claims claims = Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody();
        log.info("Token validated for user: {}, expires: {}", 
                claims.getSubject(), claims.getExpiration());
        return true;
    } catch (ExpiredJwtException e) {
        log.warn("Token expired: {}", e.getMessage());
        return false;
    } catch (SignatureException e) {
        log.error("Invalid signature: {}", e.getMessage());
        return false;
    }
}
```