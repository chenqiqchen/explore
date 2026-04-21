---
theme: channing-cyan
date: 2026-04-21
---
# 【Nginx】前端项目部署与反向代理实战指南

## 前言

很多前端同学第一次接触 Nginx，都是在项目上线前夕：

- 本地 `npm run dev` 跑得好好的
- 一到服务器就出现 404、刷新白屏、接口跨域
- HTTPS、缓存、日志也不知道从哪下手

这篇文章不讲太多抽象概念，直接用一套可落地的配置，带你完成从静态资源托管到反向代理的常见场景。

## Nginx 是什么，前端为什么要关心

一句话理解：**Nginx 是一个高性能 Web 服务器 + 反向代理服务器**。

对前端开发最直接的价值有四个：

1. 托管打包后的静态文件（`dist`）
2. 将 `/api` 请求转发到后端服务（避免跨域）
3. 处理 SPA 刷新路由 404
4. 做缓存、压缩、HTTPS，提升访问体验

## 场景一：部署 Vue/React 单页应用（解决刷新 404）

前端项目打包后，通常会得到一个 `dist` 目录。最常见的 Nginx 配置如下：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    root /var/www/my-app/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

核心是这句：

```nginx
try_files $uri $uri/ /index.html;
```

它的作用是：当访问 `/user/profile` 这种前端路由时，如果服务器找不到对应物理文件，就回退到 `index.html`，再交给前端路由系统处理。

## 场景二：反向代理后端接口（解决跨域）

开发时你可能在 Vite 里配过 `proxy`。上线后这件事通常交给 Nginx：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    root /var/www/my-app/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### `proxy_pass` 末尾斜杠非常关键

这个地方最容易踩坑：

- `location /api/` + `proxy_pass http://127.0.0.1:8080/;`
  - 请求 `/api/user` 会转发成 `/user`
- `location /api/` + `proxy_pass http://127.0.0.1:8080;`
  - 请求 `/api/user` 会转发成 `/api/user`

到底要不要保留 `/api` 前缀，取决于你的后端路由设计。

## 场景三：静态资源缓存优化（提升首屏与重复访问体验）

前端打包后，`js/css` 文件一般带 hash，适合强缓存；`index.html` 不适合缓存太久。

```nginx
location / {
    try_files $uri $uri/ /index.html;
    add_header Cache-Control "no-cache";
}

location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2?)$ {
    expires 30d;
    add_header Cache-Control "public, max-age=2592000, immutable";
}
```

推荐策略：

- `index.html`：`no-cache`，保证能及时拿到新资源引用
- 带 hash 的静态资源：长缓存，减少重复下载

## 场景四：开启 Gzip 压缩（低成本提速）

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1024;
gzip_types text/plain text/css application/javascript application/json application/xml image/svg+xml;
```

通常这几行就能明显降低 `js/css/json` 传输体积。  
如果服务器版本支持，也可以进一步考虑 Brotli（压缩率更高）。

## 场景五：HTTPS 与 HTTP 跳转

生产环境建议全站 HTTPS，并把 HTTP 自动跳转到 HTTPS：

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name your-domain.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    root /var/www/my-app/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

## 一套可直接参考的完整配置

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name your-domain.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    root /var/www/my-app/dist;
    index index.html;

    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/javascript application/json application/xml image/svg+xml;

    location / {
        try_files $uri $uri/ /index.html;
        add_header Cache-Control "no-cache";
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|svg|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
        access_log off;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 常用 Nginx 指令速查

下面这些命令，基本覆盖了日常 80% 的运维操作：

```bash
# 查看 Nginx 版本
nginx -v

# 查看详细编译参数（排查模块支持时很有用）
nginx -V

# 检查配置文件语法是否正确
nginx -t

# 打印完整配置（含 include 进来的文件）
nginx -T

# 重载配置（不中断服务）
nginx -s reload

# 快速停止
nginx -s stop

# 优雅停止（处理完当前连接再退出）
nginx -s quit
```

如果你用的是 `systemd`（大多数 Linux 发行版默认）：

```bash
# 启动 / 停止 / 重启 / 重载
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx

# 查看运行状态
sudo systemctl status nginx

# 设置开机自启
sudo systemctl enable nginx
```

日志与排查常用命令：

```bash
# 实时看错误日志
sudo tail -f /var/log/nginx/error.log

# 实时看访问日志
sudo tail -f /var/log/nginx/access.log

# 看 80/443 端口监听情况
sudo ss -lntp | grep -E ":80|:443"
```

## 常见故障排查清单

上线后有问题，优先按这个顺序排查：

1. `nginx -t`：检查配置语法是否正确
2. `systemctl reload nginx`：确认配置已重载
3. 看日志：`/var/log/nginx/access.log` 和 `/var/log/nginx/error.log`
4. 检查防火墙与安全组端口（80/443）
5. 检查后端服务是否可访问（接口代理问题）
6. 检查前端构建产物路径与 `root` 是否一致

## 写在最后

如果是前端同学，掌握 Nginx 不需要“精通运维”，但至少要掌握三件事：

- SPA 刷新 404：`try_files`
- 接口转发与跨域：`location /api` + `proxy_pass`
- 上线性能优化：缓存 + 压缩 + HTTPS

把这三件事吃透，绝大多数前端项目上线问题都能自己解决。
