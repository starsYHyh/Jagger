---
title: "将博客部署到阿里云服务器"
date: 2024-07-28
draft: false
tags: ["BLOG"]
categories: ["Learning"]
---

之前搭建hexo博客，是将其部署到github中，通过github.io来访问，但是速度感人，所以本次尝试将hugo博客部署到阿里云服务器，通过域名进行访问，本文参考了[hugo博客部署到腾讯云轻量级服务器](https://www.sulvblog.cn/posts/blog/hugo_deploy/)。
由于是我第一次使用nginx，所以如果遇到什么问题，请多海涵并且可以参考其他博客或网络资料。

在部署之前，我已经购买了阿里云服务器（配置为2核2GB）、域名（fireflyyh.top）。

## 1. 服务器端下载并安装nginx

### 1.1 安装nginx

在ubuntu环境下：
安装nginx
```bash
sudo apt install nginx
```

将nginx设置为开机启动：
```bash
sudo systemctl enable nginx
```

启动nginx：
```bash
sudo systemctl start nginx
```

### 1.2 测试nginx
查看nginx状态
```bash
sudo systemctl status nginx
```

如果没问题，则会出现：
```bash
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2024-07-26 21:46:50 CST; 1 day 13h ago
       Docs: man:nginx(8)
    Process: 164834 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 164836 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 164837 (nginx)
      Tasks: 3 (limit: 1947)
     Memory: 4.1M
        CPU: 26ms
     CGroup: /system.slice/nginx.service
             ├─164837 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─164838 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─164839 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Jul 26 21:46:50 Firefly-Aliyun systemd[1]: Starting A high performance web server and a reverse proxy server...
Jul 26 21:46:50 Firefly-Aliyun systemd[1]: Started A high performance web server and a reverse proxy server.
```

通过阿里云服务器管理面板中的安全组标签，将服务器的80端口开放。接下来访问`http://<服务器IP地址>`，如果出现了nginx页面，则代表着配置成功。

## 2. 配置nginx
进入`/etc/nginx/`，目录树如下：
```bash
.
├── conf.d
├── fastcgi.conf
├── fastcgi_params
├── koi-utf
├── koi-win
├── mime.types
├── modules-available
├── modules-enabled
│   ├── 50-mod-http-geoip2.conf -> /usr/share/nginx/modules-available/mod-http-geoip2.conf
│   ├── 50-mod-http-image-filter.conf -> /usr/share/nginx/modules-available/mod-http-image-filter.conf
│   ├── 50-mod-http-xslt-filter.conf -> /usr/share/nginx/modules-available/mod-http-xslt-filter.conf
│   ├── 50-mod-mail.conf -> /usr/share/nginx/modules-available/mod-mail.conf
│   ├── 50-mod-stream.conf -> /usr/share/nginx/modules-available/mod-stream.conf
│   └── 70-mod-stream-geoip2.conf -> /usr/share/nginx/modules-available/mod-stream-geoip2.conf
├── nginx.conf
├── proxy_params
├── scgi_params
├── sites-available
│   └── default
├── sites-enabled
│   └── default -> /etc/nginx/sites-available/default
├── snippets
│   ├── fastcgi-php.conf
│   └── snakeoil.conf
├── uwsgi_params
└── win-utf
```
根据参考博客，要在`nginx.conf`文件中编辑`Server`字段，但是在我的这个文件中，并没有出现该字段。经过查阅资料，可以在`./sites-available/default`当中编辑。