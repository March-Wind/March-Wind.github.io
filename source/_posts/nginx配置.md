---
title: nginx 基础配置
# subtitle: 没有方法的阅读源码，会很痛苦
date: 2024-10-5 15:35:00
categories: nodejs
index_img: /img/nginx-log.png
math: true
# banner_img: /img/read-code-index.png
---

## 🌐 nginx 配置

### 📁 静态资源文件配置(适用 SPA 项目)

使用 location 进行 url 映射本机资源，^~是前缀匹配，并且命中了，就不会去后面继续匹配规则了。由于我们可能有很多的静态资源项目，所以只在当前的 location 里，把匹配到 url 的部分路径替换/www/wwwroot/chat/chat_web/。比如访问`https://qunyangbang.cn/chat_web/new-chat`，在 nginx 这层就会变成访问`/www/wwwroot/chat/chat_web/new-chat`。`autoindex` 是当路径访问的资源没有时，会尝试某个路径下的 `index.html`。`try_files` 是按照制定顺序查看文件，`try_files $uri $uri/ /chat_web/index.html;`这里就是先是原路径($uri)，再尝试路径下的 index.html($uri/,因为开启了 autoindex,就是给这里用的)，最后尝试访问绝对路径(/chat_web/index.html，访问这个路径，nginx 会把他仿作新的内部请求，重新进行 location 匹配，然后就能找到资源了)。

```
server {
  ...
  ...
  ...

  location ^~/chat_web/ {
    alias /www/wwwroot/chat/chat_web/;
    autoindex on;
    try_files $uri $uri/ /chat_web/index.html;
  }
}
```

### ⚡ 静态资源性能优化

1. 🗜️ 开启 Gzip，压缩资源文件，提高传输效率

   ```
    http{
        gzip on; # 开始gzip
        gzip_min_length  1k; # 只有当文件大小 ≥ 1KB 时才进行gzip压缩
        gzip_buffers     4 16k; # 设置gzip压缩的缓冲区，使用 4 个 16KB 的缓冲区来存储压缩数据
        gzip_http_version 1.1; # 指定启用gzip压缩的HTTP协议版本，HTTP/2：也会进行gzip压缩
        gzip_comp_level 2; # 设置gzip压缩级别，1-9（1最快但压缩率最低，9最慢但压缩率最高）
        gzip_types     # text/plain application/javascript application/x-javascript text/javascript text/css application/xml; # 指定需要压缩的MIME类型
        gzip_vary on; # 添加Vary: Accept-Encoding响应头，告诉缓存服务器根据客户端是否支持压缩来缓存不同版本
        gzip_proxied   expired no-cache no-store private auth; # 在什么情况下对代理请求进行压缩(响应头包含过期时间的请求、响应头包含Cache-Control: no-cache的请求，等)
        gzip_disable   "MSIE [1-6]\."; # 禁用特定浏览器的gzip压缩，对IE 1-6版本的浏览器禁用gzip
    }
   ```

2. 💾 图片/CSS/JS 缓存

   ```
   server {

      # JS/CSS 文件的专门 location
        location ~_ ^/chat_web/._\.(js|css)$ {
        alias /www/wwwroot/chat/chat_web/;
        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
      }

      # 图片文件的专门 location
      location ~_ ^/chat_web/._\.(jpg|jpeg|png|gif|ico|svg)$ {
        alias /www/wwwroot/chat/chat_web/;
        expires 3M;
        add_header Cache-Control "public, max-age=7776000";
      }

      # 路由专用 location
      location ^~ /chat_web/ {
        alias /www/wwwroot/chat/chat_web/;
        autoindex on;
        try_files $uri $uri/ /chat_web/index.html;
      }

    }
   ```

### 🔗 后端接口服务

直接使用 proxy_pass 代理到本机的某个端口就行了。

```
  # chat项目的后端
  location ^~ /chat_server/ {
    proxy_pass http://127.0.0.1:4001/;
  }

```

### 🔒 SSL 证书配置

> SSL 证书还是挺贵的，可以使用 Let’s Encrypt 的证书证书，参考链接：https://diamondfsd.com/lets-encrytp-hand-https/#google_vignette

这里使用的是 307,因为 307 的转发可以保留所有的参数，比如 method 为 POST，301 会变成 GET 请求，307 会继续保持 POST。

```
http {

    ...
    ...

    server {
      listen 80;
      server_name qunyangbang.cn www.qunyangbang.cn m.qunyangbang.cn;
      root /www/wwwroot;
      # 重定向所有 HTTP 请求到 HTTPS
        return 307 https://$server_name$request_uri;
    }

    server {
        #SSL 默认访问端口号为 443
        listen 443 ssl;
        #请填写绑定证书的域名,这个证书是单域名的
        server_name qunyangbang.cn www.qunyangbang.cn;
        #请填写证书文件的相对路径或绝对路径
        ssl_certificate /www/server/panel/vhost/cert/qunyangbang.cn_bundle.crt;
        #请填写私钥文件的相对路径或绝对路径
        ssl_certificate_key /www/server/panel/vhost/cert/qunyangbang.cn.key;
        ssl_session_timeout 5m;
        #请按照以下协议配置
        ssl_protocols TLSv1.2 TLSv1.3;
        #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;
        location / {
            return 301 https://www.qunyangbang.cn;
            root /www/wwwroot;
            index  index.html index.htm;
        }
    }
}

```
