+++
title = '【生产问题】分享一次Nginx正向代理的需求'
date = 2020-04-17T11:26:41+08:00
draft = false
categories = [
    "Linux",
    "Nginx"
]
+++

> 最近接到了一个需求：通过 Nginx 代理把现网一个自研代理程序给替换掉，感觉有点意思，也有所收益，简单分享下。

### 需求背景

部门的生产环境异常复杂，有部分第三方引入的系统位于特殊网络隔离区域，请求这些系统需要通过 2 层网络代理，如图所示：

![12](../images/12.png)

中心源系统请求目标系统 API 的形式各异，我简单收集了下，至少有如下 3 种：
```shell
curl --digest -u admin:xxxxxx 'http://10.xxx.xxx.xxx:8080/foo/boo?Id=123456789&vId=1234' 
 
curl -d '{"eventId": 20171116, "timestamp": 123456, "caller": "XXP", "version": "1.0", "interface": {"interfaceName": "XXPVC", "para": {"detail": {"owner": "xxxxxxx"}}}, "password": "xxxxxx", "callee": "XXPVC"}' http://10.x.x.x:8080/t/api
 
curl -X PUT -H "Content-Type: application/json" -d'{"vp":{"id":"ab27adc8-xxx-xxxx-a732-fbde162ebdd3"}}' "http://10.x.x.x/v1.0/peers/show_connectioninfos"
```
<!--more-->

目前开发同事是用 lighthttp 二次开发实现了这个需求（猜测用到了一堆判断和转发逻辑），存在一定的后期维护工作量，而且这个 GG 已经转岗去其他部门了，现任开发 GG 就想直接通过 Nginx 代理来实现，淘汰这个组件，因此就将这个需求丢给了我这个运维了。

### 需求分析

拿到需求后，我分析了下，应该需要使用正向代理来实现，我们来看下普通的一级正向代理写法：
```shell
server {  
    listen  8080;  
    location / {  
        proxy_pass http://$host$request_uri; 
    }  
}
```
这个规则的意思是将所有请求都代理到请求对应的主机。这个在内网正向代理上网的时候会用到，这时候用户只需要将你提供的代理设置为 http_proxy，就可以访问到直接访问不到的站点。

看起来好像可以满足需求了，But...实际需求是要经过 2 层代理，那第一层代理的$host 必须是固定为第二层代理的地址了！而且 Nginx 也不支持类似 http_proxy 的设置，所以照搬正向代理是行不通的。

### 最终解决
