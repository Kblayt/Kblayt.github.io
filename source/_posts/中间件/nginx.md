---
title: nginx
date: 2022-10-29 12:43:50
categories:
    - 中间件
tags:
    - Jenkins
---

{% pdf Nginx_day01.pdf %}


{% pdf Nginx_day02.pdf %}


{% pdf Nginx_day03.pdf %}


{% pdf Nginx_day04.pdf %}


{% pdf Nginx_day05.pdf %}

[Nginx.xmind](C:/Users/Kblayt/Desktop/youdaonote/youdaonote-pull/youdaonote/youdaonote-attachments/WEBRESOURCEc6d2252a79f1a3de703585df478bbf8bNginx.xmind)

## 负载均衡有四种方式

- 轮询（默认）

- 权重轮询（weight）

- ip_hash（让某一客户机在相当长的一段时间内只访问固定的后端的某台真实的web服务器）

- least_conn（请求将被传递给当前拥有最少活跃连接的server，同时考虑权重weight的因素）

## 配置在niginx.conf中

①http节点下面配置

```
upstream serverPools {
	#ip_hash;
	#least_conn;
	server localhost:8080 backup;
	server localhost:8081 weight=1;
	server localhost:8082 weight=2;
	server localhost:8083 weight=3;
}

```

②server节点下配置

```
location /mystation/ {
        proxy_pass  http://serverPools/;
	#proxy_set_header Host $host;
    }

```

