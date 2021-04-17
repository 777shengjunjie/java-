---
title: nginx的负载均衡与反向代理
categories:
- shengjunjie
tags:
-  
---


<!--more-->

## 解决方案
### 前期准备
1.下载nginx
2.文件夹目录
* nginx
  * sys
  * regist

3.了解反向代理好处 
![nginx](https://i.imgur.com/4LAh5Jx.png)
4.nginx 讲解
![反向代理讲解](https://i.imgur.com/ngcdxbA.jpg)
5.负载均衡
![负载均衡](https://i.imgur.com/3AaqOx7.png)

### 核心思路

### 详细过程
```

user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    # 反向代理配置
    upstream tomcatserver1 {
             server 192.168.72.49:8081;
          }

    server {
	    listen       80; #默认端口号
	    server_name  localhost;  #域名或者ip

	    location / {
	        root   /sys;  # root 默认访问资源的目录
          # proxy_pass   http://tomcatserver1; 反向代理服务器的地址
	        index  index.html index.htm;  # 默认访问资源名称
	    }

	    error_page   500 502 503 504  /50x.html;  #错误页面
	    location = /50x.html {
	        root   /sys;
	    }
	}


    server {
	    listen       81; #默认端口号
	    server_name  localhost;  #域名或者ip

	    location / {
	      #  root   /regist;  # root 默认访问资源的目录
           proxy_pass   http://tomcatserver1; 反向代理服务器的地址
	        index  index.html index.htm;  # 默认访问资源名称
	    }

	    error_page   500 502 503 504  /50x.html;  #错误页面
	    location = /50x.html {
	        root   /sys;
	    }
	}
}

```



## 总结
一般使用第二种反向代理配置