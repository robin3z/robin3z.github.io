---
title: "记录Nginx慢请求到单独日志文件"
slug: "nginx-slow-request-log"
date: 2023-03-19T07:20:35Z
description: ""
tags: [Nginx]
---

## Nginx 记录响应时间的变量

Nginx 本身有相关变量(request_time, upstream_response_time)用来记录响应时间， 为了做性能分析，需要记录这些值到access log里。 Nginx 作为反向代理服务器，实际是将请求转发给具体的上游应用服务器，所以upstream_response_time的价值更大一些，需要重点关注。 

## 单独的慢请求日志文件

如果响应时间数据混在普通的access log日志文件里， 那么后续做性能分析时会非常不方便。 Nginx 1.7.0 版本的access_log指令新增了一个if参数，使得我们有能力根据条件判断是否记录日志。 

access_log指令的语法格式：
```
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
```

只有当if参数中的 condition 变量的值不为0或空时，Nginx才会记录日志。

我们可以使用 Map 指令生成一个新变量is_slow_request， 比如只有当变量 upstream_response_time 的值大于等于2秒时，变量 is_slow_request 的值才设置为1， 然后将 is_slow_request 传给 access_log 的 if 参数， 这样就实现了将慢请求记录到单独日志文件的功能。 以下是一个具体的 Nginx 配置例子。
```
http {
    log_format  slow  '"$request" '
                      '$status $request_length:$bytes_sent:$body_bytes_sent $upstream_response_time';

	map $upstream_response_time $is_slow_request {
	  ~^[23456789] 1; #响应时间大于等于2秒则为慢请求
	  default 0;
	}

	server {

		location / {
			proxy_pass   http://127.0.0.1:8080;
			access_log /var/log/nginx/slow.log slow if=$is_slow_request;
			access_log /var/log/nginx/all.log;
		}

	}
					  
}
```
