---
title: Spring Security核心原理：认证与授权流程详解
date: 2026-04-21 10:00:00
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring Security,认证,授权,OAuth2,JWT
description: 深入理解Spring Security的认证与授权机制，核心过滤器链解析
cover:
---

## 前言

Spring Security 是 Spring 生态中最核心的安全框架，提供完整的认证（Authentication）和授权（Authorization）解决方案。本文深入解析其核心原理。

## 核心概念

### 认证 vs 授权

| 概念 | 说明 | 示例 |
|------|------|------|
| 认证 (Authentication) | 验证你是谁 | 登录验证用户名密码 |
| 授权 (Authorization) | 验证你能做什么 | RBAC角色权限控制 |

### 核心接口

```java
// 认证
public interface Authentication {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated);
}

// 认证提供者
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication);
    boolean supports(Class<?> authentication);
}

// 用户详情
public interface UserDetails {
    String getPassword();
    String getUsername();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

---

## 过滤器链

### Spring Security过滤器链

```
┌─────────────────────────────────────────────────────────────────┐
│                      Spring Security 过滤器链                        │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. ChannelProcessingFilter                                 │  │
│  │    - 验证请求通道（HTTP/HTTPS）                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. SecurityContextPersistenceFilter                      │  │
│  │    - 从Session加载/保存SecurityContext                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. LogoutFilter                                          │  │
│  │    - 处理注销请求                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. UsernamePasswordAuthenticationFilter                   │  │
│  │    - 验证用户名密码                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 5. BasicAuthenticationFilter                             │  │
│  │    - 处理Basic认证                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 6. SecurityContextHolderAwareRequestFilter                │  │
│  │    - 包装请求为SecurityContextHolderAwareRequestWrapper  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 7. ExceptionTranslationFilter                            │  │
│  │    - 转换AccessDeniedException和AuthenticationException  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              ↓                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 8. FilterSecurityInterceptor                             │  │
│  │    - 执行授权检查                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 认证流程

### UsernamePasswordAuthenticationFilter

```java
public class UsernamePasswordAuthenticationFilter extends
        AbstractAuthenticationProcessingFilter {

    public Authentication attemptAuthentication(
            HttpServletRequest request,
            HttpServletResponse response) throws AuthenticationException {

        // 1. 获取用户名密码
        String username = obtainUsername(request);
        String password = obtainPassword(request);

        // 2. 创建认证令牌
        UsernamePasswordAuthenticationToken authRequest =
            new UsernamePasswordAuthenticationToken(username, password);

        // 3. 设置详情
        setDetails(request, authRequest);

        // 4. 认证
        return this.getAuthenticationManager().authenticate(authRequest);
    }
}
```

### AuthenticationManager

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}

// 默认实现：ProviderManager
public class ProviderManager implements AuthenticationManager {
    private List<AuthenticationProvider> providers;

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {
        for (AuthenticationProvider provider : providers) {
            if (provider.supports(authentication.getClass())) {
                result = provider.authenticate(authentication);
                if (result != null) {
                    copyDetails(authentication, result);
                    return result;
                }
            }
        }
        throw new ProviderNotFoundException();
    }
}
```

### DaoAuthenticationProvider

```java
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    private UserDetailsService userDetailsService;

    @Override
    protected void additionalAuthenticationChecks(
            UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) {

        String presentedPassword = authentication.getCredentials().toString();
        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException("密码错误");
        }
    }

    @Override
    protected UserDetails retrieveUser(
            String username,
            UsernamePasswordAuthenticationToken authentication) {

        UserDetails loadedUser = userDetailsService.loadUserByUsername(username);
        if (loadedUser == null) {
            throw new UserNotFoundException("用户不存在");
        }
        return loadedUser;
    }
}
```

---

## 授权流程

### FilterSecurityInterceptor

```java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor
        implements Filter {

    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        InterceptorStatusToken token = super.beforeInvocation(fi);
        try {
            chain.doFilter(request, response);
        } finally {
            super.afterInvocation(token, null);
        }
    }

    protected AuthorizationDecision check(AbstractIntegrationObject object) {
        AccessDecisionManager accessDecisionManager = getAccessDecisionManager();
        return accessDecisionManager.decide(auth, object, configAttributes);
    }
}
```

### 决策模式

| 模式 | 说明 |
|------|------|
| AffirmativeBased | 一个通过即通过（默认） |
| ConsensusBased | 多数通过 |
| UnanimousBased | 全票通过 |

### 投票器

```java
// RoleVoter
public class RoleVoter implements AccessDecisionVoter<Object> {
    public int vote(Authentication authentication, Object object,
                   Collection<ConfigAttribute> attributes) {
        for (ConfigAttribute attribute : attributes) {
            if (attribute.getAttribute() != null &&
                attribute.getAttribute().startsWith("ROLE_")) {
                // 检查是否具有相应角色
            }
        }
        return ACCESS_ABSTAIN;
    }
}

// WebExpressionVoter
public class WebExpressionVoter implements AccessDecisionVoter<FilterInvocation> {
    public int vote(Authentication authentication, FilterInvocation fi,
                   Collection<ConfigAttribute> definition) {
        WebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        WebSecurityExpressionRoot root = handler.createSecurityExpressionRoot(
            authentication, fi);
        return evaluate(expression) ? ACCESS_GRANTED : ACCESS_DENIED;
    }
}
```

---

## Spring Security配置

### Java配置

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .antMatchers("/public/**").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .antMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
            )
            .httpBasic(basic -> basic.realmName("MyApp"))
            .csrf(csrf -> csrf.disable());  // API场景通常禁用
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
            .inMemoryAuthentication()
                .withUser("user").password("{noop}password").roles("USER")
                .and()
                .withUser("admin").password("{noop}admin").roles("USER", "ADMIN");
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 自定义UserDetailsService

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("用户不存在"));

        return User.builder()
            .username(user.getUsername())
            .password(user.getPassword())
            .authorities(user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
                .toArray(GrantedAuthority[]::new))
            .accountExpired(false)
            .accountLocked(user.isLocked())
            .credentialsExpired(false)
            .disabled(!user.isEnabled())
            .build();
    }
}
```

---

## JWT认证

### JWT结构

```
┌─────────────────────────────────────────────────────────────────┐
│                         JWT 结构                                   │
│                                                                   │
│  Header.Payload.Signature                                         │
│                                                                   │
│  Header:                                                          │
│  {                                                                │
│    "alg": "HS256",                                               │
│    "typ": "JWT"                                                   │
│  }                                                                │
│                                                                   │
│  Payload:                                                         │
│  {                                                                │
│    "sub": "1234567890",                                          │
│    "name": "John Doe",                                           │
│    "roles": ["USER", "ADMIN"],                                    │
│    "iat": 1516239022,                                           │
│    "exp": 1516242622                                             │
│  }                                                                │
│                                                                   │
│  Signature:                                                        │
│  HMACSHA256(                                                     │
│    base64UrlEncode(header) + "." + base64UrlEncode(payload),    │
│    secret                                                         │
│  )                                                                │
└─────────────────────────────────────────────────────────────────┘
```

### JWT过滤器

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenProvider tokenProvider;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain)
            throws ServletException, IOException {

        String token = getJwtFromRequest(request);

        if (StringUtils.hasText(token) && tokenProvider.validateToken(token)) {
            String username = tokenProvider.getUsernameFromToken(token);

            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());

            authentication.setDetails(
                new AuthenticationDetailsSource<HttpServletRequest>()
                    .buildDetails(request));

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
```

### JWT工具类

```java
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Value("${jwt.expiration}")
    private long jwtExpiration;

    public String generateToken(Authentication authentication) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()))
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }

    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
        return claims.getSubject();
    }

    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

---

## OAuth2配置

### OAuth2资源服务器

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) {
        resources.tokenStore(tokenStore());
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            );
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey("secret");
        return converter;
    }
}
```

---

## 常见面试题

**Q1：Spring Security的过滤器链顺序？**

> 答：ChannelProcessingFilter → SecurityContextPersistenceFilter → LogoutFilter → UsernamePasswordAuthenticationFilter → BasicAuthenticationFilter → ... → FilterSecurityInterceptor。

**Q2：Authentication和Authorization的区别？**

> 答：Authentication是认证，验证身份（你是谁）；Authorization是授权，验证权限（你能做什么）。

**Q3：UserDetailsService的作用？**

> 答：加载用户信息用于认证。需要实现loadUserByUsername方法，返回UserDetails对象。

**Q4：JWT的结构？**

> 答：Header.Payload.Signature。Header包含算法，Payload包含声明（用户名、角色、过期时间等），Signature用于验证。

**Q5：Spring Security如何实现"记住我"功能？**

> 答：通过RememberMeAuthenticationFilter实现，使用TokenBasedRememberMeServices或PersistentTokenRepository。

---

## 总结

Spring Security是完整的安全解决方案：
- **过滤器链**：按顺序执行的安全过滤器
- **认证**：Authentication接口、UserDetailsService
- **授权**：FilterSecurityInterceptor、AccessDecisionManager
- **JWT**：无状态认证方案
- **OAuth2**：第三方授权解决方案
