---
title: Spring MVC请求处理流程：DispatcherServlet源码解析
date: 2023-07-17 11:33:49
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring MVC,DispatcherServlet,请求处理,HandlerMapping
description: 深入理解Spring MVC请求处理流程，DispatcherServlet核心组件解析
cover:
---

## 前言

Spring MVC 是 Java Web 开发最常用的框架。理解它的请求处理流程，是排查问题和深入学习的基础。本文通过源码解析 Spring MVC 的请求处理链路。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Spring MVC 请求处理                          │
│                                                                   │
│  ┌─────────┐     ┌─────────────┐     ┌──────────┐              │
│  │  用户    │ ──▶ │Dispatcher  │ ──▶ │ Handler  │              │
│  │  请求    │     │  Servlet   │     │ Mapping  │              │
│  └─────────┘     └─────────────┘     └──────────┘              │
│                        │                     │                     │
│                        ▼                     ▼                     │
│                 ┌─────────────┐     ┌──────────┐              │
│                 │ HandlerAdapter│     │ Handler  │              │
│                 └─────────────┘     └──────────┘              │
│                        │                     │                     │
│                        ▼                     ▼                     │
│                 ┌─────────────┐     ┌──────────┐              │
│                 │ ViewResolver│     │Controller│              │
│                 └─────────────┘     └──────────┘              │
│                        │                     │                     │
│                        ▼                     ▼                     │
│                    ┌─────────┐          ┌─────────┐             │
│                    │ View    │          │  Model  │             │
│                    └─────────┘          └─────────┘             │
│                        │                                        │
│                        ▼                                        │
│                    ┌─────────┐                                  │
│                    │ 响应    │                                  │
│                    └─────────┘                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心组件

### 组件职责

| 组件 | 职责 |
|------|------|
| DispatcherServlet | 前端控制器，请求入口 |
| HandlerMapping | 映射请求到具体的Handler |
| HandlerAdapter | 执行Handler，适配不同类型 |
| Controller | 处理请求的业务逻辑 |
| ViewResolver | 解析视图名称到View |
| View | 渲染视图，生成响应 |

---

## DispatcherServlet

### 继承体系

```java
public class DispatcherServlet extends FrameworkServlet {

    // 核心组件
    private MultipartResolver multipartResolver;
    private LocaleResolver localeResolver;
    private ThemeResolver themeResolver;
    private List<HandlerMapping> handlerMappings;
    private List<HandlerAdapter> handlerAdapters;
    private List<ViewResolver> viewResolvers;
}
```

### 请求入口

```java
public class DispatcherServlet extends FrameworkServlet {

    protected void doDispatch(HttpServletRequest request,
                             HttpServletResponse response) throws Exception {

        // 1. 获取Handler
        HandlerExecutionChain handler = getHandler(request);
        if (handler == null) {
            noHandlerFound(request, response);
            return;
        }

        // 2. 获取HandlerAdapter
        HandlerAdapter ha = getHandlerAdapter(handler.getHandler());

        // 3. 执行拦截器前置处理
        if (!handler.applyPreHandle(request, response)) {
            return;
        }

        // 4. 执行Controller
        ModelAndView mv = ha.handle(request, response, handler.getHandler());

        // 5. 执行拦截器后置处理
        handler.applyPostHandle(request, response, mv);

        // 6. 处理视图渲染
        processDispatchResult(request, response, mv);
    }
}
```

---

## 请求处理流程

### 详细流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      请求处理详细流程                               │
│                                                                   │
│  1. 用户发送请求到 DispatcherServlet                             │
│                              │                                     │
│  2. DispatcherServlet 调用 HandlerMapping                       │
│     getHandler(request) ──▶ 遍历 handlerMappings                 │
│                              │                                     │
│  3. 匹配到 @RequestMapping("/user")                             │
│                              │                                     │
│  4. 返回 HandlerExecutionChain (Handler + Interceptors)         │
│                              │                                     │
│  5. DispatcherServlet 调用 HandlerAdapter                       │
│     ha.handle(request, response, handler)                        │
│                              │                                     │
│  6. HandlerAdapter 执行 Controller                              │
│     └─ controller.method(request, response)                      │
│                              │                                     │
│  7. 返回 ModelAndView (model + viewName)                        │
│                              │                                     │
│  8. DispatcherServlet 调用 ViewResolver                         │
│     viewResolver.resolveViewName(viewName, locale)               │
│                              │                                     │
│  9. 返回 View 对象                                              │
│                              │                                     │
│  10. View.render(model, request, response)                     │
│                              │                                     │
│  11. 渲染HTML/JSON响应                                          │
│                              │                                     │
│  12. 响应客户端                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## HandlerMapping

### 组件接口

```java
public interface HandlerMapping {
    // 根据请求获取处理器链
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

### RequestMappingHandlerMapping

```java
// 处理 @RequestMapping 注解
@Component
public class RequestMappingHandlerMapping implements HandlerMapping {

    @Override
    public HandlerExecutionChain getHandler(HttpServletRequest request)
            throws Exception {

        // 1. 获取请求方法、路径
        String lookupPath = urlPathHelper.getLookupPathForRequest(request);
        RequestMappingInfo info = createRequestMappingInfo(request);

        // 2. 匹配 @RequestMapping 注解的方法
        RequestMappingInfo matchingInfo = handlerMapping.getMatchingMapping(info, request);
        if (matchingInfo == null) {
            return null;
        }

        // 3. 获取Handler和Interceptor
        HandlerMethod handlerMethod = handlerMethods.get(matchingInfo);
        List<InterceptorRegistration> interceptors = getInterceptors(info);

        // 4. 构建执行链
        return new HandlerExecutionChain(handlerMethod, interceptors);
    }
}
```

### 常用注解

```java
@Controller
@RequestMapping("/api")
public class UserController {

    @RequestMapping(value = "/user/{id}", method = RequestMethod.GET)
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping("/user")
    public User createUser(@RequestBody @Valid User user) {
        return userService.save(user);
    }

    @GetMapping("/user")
    public List<User> listUsers(@RequestParam(defaultValue = "1") int page,
                                @RequestParam(defaultValue = "10") int size) {
        return userService.findAll(page, size);
    }
}
```

---

## HandlerAdapter

### 组件接口

```java
public interface HandlerAdapter {
    // 是否支持此Handler
    boolean supports(Object handler);

    // 执行Handler，返回ModelAndView
    ModelAndView handle(HttpServletRequest request,
                       HttpServletResponse response,
                       Object handler) throws Exception;
}
```

### RequestMappingHandlerAdapter

```java
public class RequestMappingHandlerAdapter implements HandlerAdapter {

    @Override
    public ModelAndView handle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {

        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // 1. 解析参数
        Object[] args = resolveParameters(request, response, handlerMethod);

        // 2. 执行目标方法
        Object result = handlerMethod.invoke(args);

        // 3. 处理返回值
        ModelAndView mv = handleReturnValue(result, handlerMethod);

        return mv;
    }

    // 参数解析
    private Object[] resolveParameters(HttpServletRequest request,
                                       HttpServletResponse response,
                                       HandlerMethod handlerMethod) {

        // @PathVariable、@RequestParam、@RequestBody 等
        // 遍历参数注解，调用对应的 ArgumentResolver
    }

    // 返回值处理
    private ModelAndView handleReturnValue(Object returnValue,
                                           HandlerMethod handlerMethod) {

        // @ResponseBody  → 直接写入响应
        // 返回ViewName  → ModelAndView
        // 返回String    → ViewName
    }
}
```

---

## 参数绑定

### 常用参数注解

```java
@Controller
public class UserController {

    // @PathVariable: URL路径参数
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable("id") Long id) {
        return userService.findById(id);
    }

    // @RequestParam: 请求参数
    @GetMapping("/user")
    public List<User> list(@RequestParam(defaultValue = "1") int page,
                           @RequestParam(required = false) String name) {
        return userService.findByName(name, page);
    }

    // @RequestBody: 请求体JSON
    @PostMapping("/user")
    public User create(@RequestBody User user) {
        return userService.save(user);
    }

    // @RequestHeader: 请求头
    @GetMapping("/user/{id}")
    public User get(@PathVariable Long id,
                    @RequestHeader("Authorization") String token) {
        return userService.findById(id);
    }

    // @CookieValue: Cookie值
    @GetMapping("/user/{id}")
    public User get(@PathVariable Long id,
                    @CookieValue("SESSIONID") String sessionId) {
        return userService.findById(id);
    }

    // @ModelAttribute: 模型属性
    @PostMapping("/user/update")
    public User update(@ModelAttribute User user) {
        return userService.update(user);
    }
}
```

### 参数绑定原理

```java
// 参数解析器
public interface HandlerMethodArgumentResolver {
    boolean supportsParameter(MethodParameter parameter);
    Object resolveArgument(MethodParameter parameter, ...) throws Exception;
}

// 例如 @RequestParam 处理
public class RequestParamMethodArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(RequestParam.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ...) {
        RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
        String paramName = ann.value();
        return request.getParameter(paramName);
    }
}
```

---

## 拦截器（HandlerInterceptor）

### 接口定义

```java
public interface HandlerInterceptor {

    // 前置处理，返回true继续执行，false中断
    default boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        return true;
    }

    // 后置处理
    default void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler,
                           ModelAndView modelAndView) throws Exception {
    }

    // 视图渲染完成后（finally）
    default void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler,
                               Exception ex) throws Exception {
    }
}
```

### 执行链

```java
public class HandlerExecutionChain {

    private final Object handler;
    private final HandlerInterceptor[] interceptors;

    // 前置处理
    public boolean applyPreHandle(HttpServletRequest request,
                                 HttpServletResponse response) throws Exception {
        for (int i = 0; i < interceptors.length; i++) {
            if (!interceptors[i].preHandle(request, response, handler)) {
                return false;
            }
        }
        return true;
    }

    // 后置处理
    public void applyPostHandle(HttpServletRequest request,
                               HttpServletResponse response,
                               ModelAndView mv) throws Exception {
        for (int i = interceptors.length - 1; i >= 0; i--) {
            interceptors[i].postHandle(request, response, handler, mv);
        }
    }
}
```

### 使用示例

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || !tokenService.validate(token)) {
            response.setStatus(401);
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request,
                          HttpServletResponse response,
                          Object handler,
                          ModelAndView mv) throws Exception {
        // 设置通用响应头
        response.setHeader("X-Response-Time", "OK");
    }
}
```

---

## ViewResolver

### 视图解析流程

```java
public class DispatcherServlet {

    private void processDispatchResult(HttpServletRequest request,
                                      HttpServletResponse response,
                                      ModelAndView mv) throws Exception {

        // 1. 没有视图，直接返回
        if (mv == null) {
            return;
        }

        // 2. 获取ViewResolver
        ViewResolver viewResolver = getViewResolver();
        View view;
        String viewName = mv.getViewName();

        // 3. 解析视图名
        view = viewResolver.resolveViewName(viewName, Locale.getDefault());

        // 4. 渲染视图
        view.render(mv.getModel(), request, response);
    }
}
```

### JSP视图

```java
// InternalResourceViewResolver 配置
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
    }
}
```

### Thymeleaf视图

```java
@Configuration
public class ThymeleafConfig {

    @Bean
    public ThymeleafViewResolver thymeleafViewResolver() {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine());
        resolver.setSuffix(".html");
        resolver.setPrefix("classpath:/templates/");
        return resolver;
    }
}
```

---

## 异常处理

### @ExceptionHandler

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(value = {BusinessException.class})
    @ResponseBody
    public Result handleBusinessException(BusinessException e) {
        return Result.error(e.getMessage());
    }

    @ExceptionHandler(value = {Exception.class})
    @ResponseBody
    public Result handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error("系统繁忙");
    }
}
```

### SimpleMappingExceptionResolver

```java
@Configuration
public class ExceptionConfig {

    @Bean
    public SimpleMappingExceptionResolver exceptionResolver() {
        SimpleMappingExceptionResolver resolver = new SimpleMappingExceptionResolver();
        Properties mappings = new Properties();
        mappings.setProperty("BusinessException", "error/business");
        mappings.setProperty("SystemException", "error/system");
        resolver.setExceptionMappings(mappings);
        return resolver;
    }
}
```

---

## 面试高频问题

**Q1：Spring MVC请求处理流程？**

> 答：请求→DispatcherServlet→HandlerMapping匹配Handler→HandlerAdapter执行Controller→返回ModelAndView→ViewResolver解析视图→View渲染→响应。

**Q2：HandlerMapping和HandlerAdapter的区别？**

> 答：HandlerMapping负责将请求映射到具体的Controller方法；HandlerAdapter负责执行Controller方法并处理参数和返回值。

**Q3：@RequestBody和@ResponseBody的区别？**

> 答：@RequestBody将请求体JSON反序列化为Java对象；@ResponseBody将返回值序列化为JSON写入响应体。

**Q4：拦截器和过滤器的区别？**

> 答：过滤器是Servlet规范，作用于请求进入Servlet之前；拦截器是Spring MVC组件，作用于Handler执行链前后。

**Q5：如何自定义参数解析器？**

> 答：实现HandlerMethodArgumentResolver接口，定义supportsParameter和resolveArgument方法，通过@Bean注册。

---

## 总结

Spring MVC请求处理流程清晰解耦：
- **DispatcherServlet**：前端控制器，请求统一入口
- **HandlerMapping**：映射请求到Handler
- **HandlerAdapter**：执行Handler
- **ViewResolver**：解析视图名到View
- **View**：渲染视图
- **异常处理**：@ExceptionHandler统一处理
