# Nginx负载均衡：算法、配置与高级特性

## 前言

Nginx是高性能的HTTP服务器和反向代理服务器，因其高并发、低内存消耗而广泛应用于互联网架构中。负载均衡是Nginx最核心的功能之一，本文将深入探讨Nginx的负载均衡算法、配置方法以及健康检查、故障转移等高级特性。

## Nginx架构

### 进程模型

```
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      Master Process                             │
│                    (管理进程，控制启动/重载)                       │
│                            │                                    │
│              ┌─────────────┴─────────────┐                      │
│              ▼                           ▼                       │
│     ┌─────────────────┐         ┌─────────────────┐             │
│     │  Worker Process │         │  Worker Process │             │
│     │     (工作进程)   │         │     (工作进程)   │             │
│     │   处理请求       │         │   处理请求       │             │
│     └─────────────────┘         └─────────────────┘             │
│              │                           │                       │
│              └─────────────┬─────────────┘                      │
│                              ▼                                    │
│                     ┌─────────────────┐                         │
│                     │   连接池         │                         │
│                     │  (连接复用)       │                         │
│                     └─────────────────┘                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

配置重载流程：
1. Master进程接收SIGHUP信号
2. 启动新的Worker进程
3. 旧Worker处理完当前请求后退出
4. 新Worker开始接收请求
```

### 事件驱动模型

```c
// Nginx使用事件驱动模型，非阻塞IO

// 多路复用技术选择：
// - Linux: epoll
// - FreeBSD: kqueue
// - Solaris: /dev/poll

// 工作流程：
// 1. Worker进程监听多个socket
// 2. 某个socket就绪（可读/可写）
// 3. epoll返回该socket
// 4. Worker处理请求（非阻塞）
// 5. 处理完继续处理其他事件
```

## 负载均衡算法

### 1. 轮询（Round Robin）

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}

# 默认配置即为轮询
# 每个请求依次分配给后端服务器
# 请求分布：101 -> 102 -> 103 -> 101 -> 102 -> 103 ...
```

### 2. 加权轮询（Weighted Round Robin）

```nginx
upstream backend {
    # weight表示权重，默认为1
    # 服务器性能越好，权重越高
    server 192.168.1.101:8080 weight=5;   # 权重5
    server 192.168.1.102:8080 weight=2;    # 权重2
    server 192.168.1.103:8080 weight=3;    # 权重3
}

# 请求分布：按权重比例
# 101, 101, 101, 101, 101, 102, 102, 103, 103, 103
# 每10个请求：5个到101，2个到102，3个到103
```

### 3. 最少连接（Least Connections）

```nginx
upstream backend {
    least_conn;  # 最少连接算法

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

# 将请求分配给当前连接数最少的服务器
# 适用于请求处理时间差异较大的场景
```

### 4. IP哈希（IP Hash）

```nginx
upstream backend {
    ip_hash;  # IP哈希算法

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

# 相同IP的请求总是发送到同一台服务器
# 优点：Session会话保持
# 缺点：服务器宕机时，可能导致session丢失
```

### 5. URL哈希（Hash）

```nginx
upstream backend {
    # 按URL的hash值分配请求
    hash $request_uri consistent;

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

# consistent: ketama一致性哈希算法
# 优点：服务器变化时，只影响少部分请求
# 适用于缓存服务器场景
```

### 6. 响应时间加权（Least Time - Nginx Plus）

```nginx
# Nginx Plus独有功能
upstream backend {
    least_time header;  # 基于header响应时间
    # 或 least_time last_byte;

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}
```

## 完整配置示例

### HTTP负载均衡

```nginx
# /etc/nginx/nginx.conf

# 运行用户
user nginx;

# 工作进程数，通常设置为CPU核心数
worker_processes auto;

# 错误日志
error_log /var/log/nginx/error.log warn;

# 进程ID
pid /var/run/nginx.pid;

events {
    # 每个worker的最大连接数
    worker_connections 10240;

    # 使用epoll（Linux）
    use epoll;

    # 允许multi_accept
    multi_accept on;
}

http {
    # 引入mime类型
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'upstream: $upstream_addr '
                    'upstream_status: $upstream_status '
                    'request_time: $request_time';

    access_log /var/log/nginx/access.log main;

    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # 连接超时
    keepalive_timeout 65;
    keepalive_requests 1000;

    # Gzip压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # 上游服务器组
    upstream backend {
        # 完整配置
        server 192.168.1.101:8080 weight=5 max_fails=3 fail_timeout=30s;
        server 192.168.1.102:8080 weight=2 max_fails=3 fail_timeout=30s;
        server 192.168.1.103:8080 weight=3 backup;  # 备份服务器

        # 使用IP哈希，会话保持
        # ip_hash;

        # 最少连接
        # least_conn;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;

            # 代理配置
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 超时配置
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # 缓冲配置
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            proxy_busy_buffers_size 8k;
        }
    }
}
```

### HTTPS负载均衡

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 健康检查

### 被动健康检查

```nginx
upstream backend {
    server 192.168.1.101:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.102:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.103:8080 max_fails=3 fail_timeout=30s;
}

# max_fails: 最大失败次数，超过后认为服务器不可用
# fail_timeout: 失败后持续不可用的时间

# 失败场景：
# 1. 连接超时
# 2. 连接被拒绝
# 3. 返回错误码（500, 502, 503, 504）
```

### 主动健康检查（Nginx Plus）

```nginx
# Nginx Plus配置
upstream backend {
    zone backend 64k;

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;
}

location / {
    proxy_pass http://backend;

    # 健康检查配置
    health_check interval=5s fails=3 passes=2 uri=/health.html;
}
```

### 第三方模块健康检查

```nginx
# 使用nginx_upstream_check_module
# 需要编译安装

upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080;

    # 健康检查配置
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

## 故障转移

### 自动故障转移

```nginx
upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
    server 192.168.1.103:8080 backup;  # 备用服务器
}

# 正常工作流程：
# 1. 请求发送到server 101
# 2. 如果101不可用，自动切换到102
# 3. 如果102也不可用，使用103
# 4. 如果都不行，使用backup服务器

# backup服务器只在主服务器全部不可用时启用
```

### 灰度发布

```nginx
# 基于IP的灰度发布
upstream backend_v1 {
    server 192.168.1.101:8080;  # 旧版本
}
upstream backend_v2 {
    server 192.168.1.102:8080;  # 新版本
}

# 10%的流量到新版本
geo $version {
    default v1;
    10.0.0.0/8 v2;
    172.16.0.0/12 v2;
    192.168.0.0/16 v2;
}

server {
    listen 80;
    server_name example.com;

    if ($version = v2) {
        proxy_pass http://backend_v2;
    }
    proxy_pass http://backend_v1;
}
```

### 基于Cookie的灰度

```nginx
# Cookie中携带版本信息
upstream backend_v1 {
    server 192.168.1.101:8080;
}
upstream backend_v2 {
    server 192.168.1.102:8080;
}

server {
    listen 80;

    # 默认路由到v1
    set $backend "backend_v1";

    # 如果cookie中version=v2，路由到v2
    if ($http_cookie ~* "version=v2") {
        set $backend "backend_v2";
    }

    proxy_pass http://$backend;
}
```

## 高级特性

### 连接池配置

```nginx
upstream backend {
    # 保持连接到上游服务器
    keepalive 32;       # 保持连接数

    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

location / {
    proxy_pass http://backend;

    # 启用keepalive
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

### 熔断机制

```nginx
# Nginx+Lua实现熔断

upstream backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

# 使用limit_req实现简单的限流
limit_req_zone $binary_remote_addr zone=backend_limit:10m rate=1000r/s;

location / {
    # 错误率超过50%时触发熔断
    limit_req zone=backend_limit burst=100 nodelay;

    proxy_pass http://backend;

    # 错误时返回503
    proxy_intercept_errors on;
    error_page 500 502 503 504 = @fallback;
}

location @fallback {
    return 503 "Service temporarily unavailable";
}
```

### WebSocket支持

```nginx
upstream websocket_backend {
    server 192.168.1.101:8080;
    server 192.168.1.102:8080;
}

server {
    listen 80;
    server_name ws.example.com;

    location / {
        proxy_pass http://websocket_backend;

        # WebSocket支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        # 超时配置
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### 动态上游配置（Nginx Plus）

```nginx
# Nginx Plus支持动态修改upstream，无需重载

# 添加新服务器
curl -X POST -d '{"server":"192.168.1.104:8080"}' \
    http://localhost:8080/upstreams/backend

# 删除服务器
curl -X DELETE \
    http://localhost:8080/upstreams/backend/192.168.1.101:8080

# 下线服务器
curl -X PUT -d '{"down":1}' \
    http://localhost:8080/upstreams/backend/192.168.1.101:8080

# 上线服务器
curl -X PUT -d '{"down":0}' \
    http://localhost:8080/upstreams/backend/192.168.1.101:8080
```

## 监控与日志

### 状态监控

```nginx
# 需要Nginx Plus或nginx-module-nginxStatus
server {
    listen 8080;
    server_name localhost;

    location /nginx_status {
        stub_status on;
        access_log off;
    }
}

# 返回：
# Active connections: 291
# server accepts handled requests
# 16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106
```

### 错误日志分析

```bash
# 统计502错误
grep "502" /var/log/nginx/access.log | wc -l

# 统计upstream响应时间
awk '{print $NF}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 统计最慢的请求
awk -F'request_time:' '{if($2)print $2}' /var/log/nginx/access.log | sort -rn | head -20
```

## 常见面试题

**Q1：Nginx的负载均衡算法有哪些？**

> 参考答案：1）轮询（Round Robin），默认算法，按顺序分配请求；2）加权轮询，根据权重比例分配；3）最少连接（Least Connections），分配给连接数最少的服务器；4）IP哈希（IP Hash），相同IP的请求分配给同一服务器，用于会话保持；5）URL哈希，按URL的hash值分配；6）响应时间加权（Least Time），Nginx Plus独有，分配给响应时间最短的服务器。

**Q2：如何保证Nginx的高可用？**

> 参考答案：1）使用Keepalived实现Nginx的主备架构；2）配置健康检查，自动故障转移；3）设置backup服务器；4）监控Nginx状态（Active connections、Reading、Writing、Waiting）；5）合理设置worker进程数和连接数；6）配置错误日志和access日志，及时发现问题。

**Q3：Nginx和Apache的区别是什么？**

> 参考答案：1）架构：Nginx使用事件驱动的异步非阻塞模型，Apache使用传统的多进程/多线程模型；2）并发处理：Nginx能处理高并发（万级），Apache相对较弱；3）资源消耗：Nginx内存消耗低，Apache较高；4）功能：Apache模块更丰富，.htaccess支持更好；5）动态内容：Apache直接支持PHP等，Nginx需要通过FastCGI。

**Q4：如何排查Nginx负载均衡的问题？**

> 参考答案：1）检查upstream配置是否正确；2）查看error_log定位错误；3）使用nginx_status查看连接状态；4）检查后端服务器是否正常响应；5）检查网络连通性；6）查看upstream响应时间和状态码；7）确认是否触发max_fails导致服务器被标记为不可用。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                      Nginx负载均衡核心                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  算法：                                                           │
│  ├─ 轮询/加权轮询          默认                                   │
│  ├─ 最少连接              长时间连接场景                          │
│  ├─ IP哈希                会话保持                                │
│  └─ URL哈希/一致性哈希     缓存场景                                │
│                                                                │
│  配置要点：                                                        │
│  ├─ upstream正确配置server                                         │
│  ├─ proxy_pass转发请求                                             │
│  ├─ 正确设置proxy_set_header                                        │
│  └─ 配置合理的超时时间                                             │
│                                                                │
│  高可用：                                                           │
│  ├─ max_fails + fail_timeout                                       │
│  ├─ backup服务器                                                  │
│  ├─ Keepalived主备                                                │
│  └─ 健康检查                                                      │
│                                                                │
│  性能优化：                                                        │
│  ├─ worker进程数 = CPU核心数                                       │
│  ├─ worker_connections合理设置                                     │
│  ├─ 开启keepalive连接复用                                         │
│  └─ 开启gzip压缩                                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

Nginx是现代互联网架构中不可或缺的组件，熟练掌握其负载均衡配置对于构建高可用、高性能系统至关重要。
