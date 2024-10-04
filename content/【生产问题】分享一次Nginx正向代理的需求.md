+++
title = '【生产问题】分享一次Nginx正向代理的需求'
date = 2020-04-17T11:26:41+08:00
draft = false
categories = [
    "Linux",
    "Nginx",
    "生产问题"
]
+++

> 最近接到了一个需求：通过 Nginx 代理把现网一个自研代理程序给替换掉，感觉有点意思，也有所收益，简单分享下。

### 需求背景

部门的生产环境异常复杂，有部分第三方引入的系统位于特殊网络隔离区域，请求这些系统需要通过 2 层网络代理，如图所示：

![12](/images/12.png)

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

既然正向代理涉及到自动提取目标主机、端口以及请求的特性，那我们就自己设计一个请求方式，方便使用 Nginx 自带规则来提取并自动代理。

我和开发约定了一个请求方式（之前也用了类似约定），方便 Nginx 来提取变量并自动代理：
```shell
curl --digest -u admin:xxxxx 'http://10.x.x.x/?proxy_schema=http&proxy_host=x.x.x.x:8080&proxy_url=/XXX/api?tId=123456&fooid=1234'
```
将真正需要请求的 API 拆成： ?schema=http&host=主机:端口&proxy_url=请求路径及参数，然后请求到第一级 Nginx 代理服务，一级代理将请求原样传给 Nginx 二级代理，然后在二级代理上通过正则提取 schema、host 和 proxy_url，并代理请求，即可满足需求。

Nginx 一级代理规则（反向代理）：反向代理到 2 个二级代理
```shell
upstream proxy_svr { 
    server 192.168.2.100:8080; 
}
server {  
    listen  8080;  
    access_log /data/wwwlogs/access.log access;
    location / {
        proxy_pass http://proxy_svr$request_uri;
   } 
 }
```
Nginx 二级代理规则（正向代理）：自动提取 url 里面约定的协议、目标主机和 url 并代理

```shell
server {  
    listen  8080;  
    #resolver 223.5.5.5; # 如果被代理的地址存在域名，需要加一个 dns 配置，否则会 502，报错信息为：no resolver defined to resolve xxx.com
    access_log /data/wwwlogs/access.log access;
    set $proxy_schema 'http';
    set $proxy_host '';
    set $proxy_url '';
    # 提取请求中的 schema 值：
    if ( $request_uri ~ (proxy_schema=([^&]+))){
        set $proxy_schema $2;
    }
    # 提取请求中的 host 值：
    if ( $request_uri ~ (proxy_host=([^&]+))){
        set $proxy_host $2;
    }
    # 提取请求中的 proxy_url 值：
    if ( $request_uri ~ (proxy_url=(.*)$)){
        set $proxy_url $2;
    }
    # 如果没能提取到则返回 404
    if ($proxy_url = '') {
        return 404;
    }
    if ($proxy_host = '') {
        return 404;
    }
    # 将提取到的请求请求转发到提取到的主机上
    location / {
       # 其他 proxy 优化参数略..
       proxy_pass $proxy_schema://$proxy_host$proxy_url;
    }  
}
```
最后再套了一层负载均衡，最终生产环境的拓扑如下：

