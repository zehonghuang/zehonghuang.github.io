+++
title = '【生产问题】时隔大半年，分享一次Nginx反向代理的需求'
date = 2020-11-13T01:03:28+08:00
draft = false
categories = [
    "Linux",
    "Nginx",
    "生产问题"
]
+++

> 博客前面分享了一篇[《分享一个 Nginx 正向代理的另类应用案例》](https://huangzehong.me/%E7%94%9F%E4%BA%A7%E9%97%AE%E9%A2%98%E5%88%86%E4%BA%AB%E4%B8%80%E6%AC%A1nginx%E6%AD%A3%E5%90%91%E4%BB%A3%E7%90%86%E7%9A%84%E9%9C%80%E6%B1%82/)，时隔不久，身为救火队员、万金油的博主又再一次接到了一个奇葩需求：


> 场景和上次有些类似，也是部门引进的第三方应用，部署在各个网络区域，从 OA 办公区域无法直接访问。目前，运营人员都需要登陆 Windows 跳板机，才能打开这些应用的 WEB 控制台。既不方便，而且还有一定 Windows 服务器的维护工作量，于是找到我们团队，希望通过运维手段来解决。

拿到这个需求后，我先问了下各个应用的基本情况，得知每个应用的框架基本是一样的，都是通过 IP+端口直接访问，页面 path 也基本一样，没有唯一性。然后拿到了一个应用 WEB 控制台地址看了下，发现 html 引用的地址都是相对路径。

乍一想，这用 Nginx 代理不好弄吧？页面 path 一样，没法根据 location 来反代到不同的后端，只能通过不同 Nginx 端口来区分，那就太麻烦了！每次他们新上一个应用，我们就得多加一个新端口来映射，这种的尾大不掉、绵绵不绝事情坚决不干，Say pass。

再一想，我就想到了上次那个正向代理另类应用方案，感觉可以拿过来改改做动态代理。原理也简单：先和用户约定一个访问形式，比如:

> Nginx 代理地址为 myproxy.oa.com，需要代理到 IP 为 192.168.2.100:8080 的控制器，用户需要访问 http://myproxy.oa.com/192.168.2.100:8080/path。
<!--more-->

动态代理的原理及实现：

> 1. Nginx 从$request_uri 变量中，通过斜杠提取第一段作为要反向代理的对象，即 proxy_pass，提取后面的作为需要反代的路径；
> 
> 2. 对于 Html、JS、CSS 等资源引用则需要通过 Nginx 的替换模块，将路径替换为上述约定形式，比如：Html 里面的 href="/js/jquery.min.js"，需要替换为 href="/goto/aHR0cDovL215cHJveHkub2EuY29tLzE5Mi4xNjguMi4xMDA6ODA4MC9qcy9qdXFlcnkubWluLmpz" target="_blank"，即所有资源地址都保证符合代理约定的形式，才能够正确走代理获取。
> 
> 3. 通过代理去访问，查看浏览器开发者工具中的 Network 和 console，找到无法访问的地址，并分析引用的位置，写规则替换掉即可。\
> 
> 4. 实际测试发现，他们的应用可能还会有 https 协议（...），而且是伪证书模式，只是开了 https 协议访问。上述设计模式下，https 页面是无法打开的，这里需要兼容 https 后端才行，因此最后的约定形式简单修改为：如果是 http://myproxy.oa.com/https-192.168.2.101:8080/path, 即：如果是 https 协议，需要在第一段 path 的 IP 前面加上 https-来区分，而 http 协议则可加可不加。

基本通过上述几步，就完全搞定了，最终 Nginx 规则如下：

```shell
server {
        listen 80;
        server_name myproxy.oa.com;
        #resolver 223.5.5.5; # 如果被代理的地址存在域名，需要加一个 dns 配置，否则会 502，报错信息为：no resolver defined to resolve xxx.com
        index index.html;
        root /html;
        # 初始化变量
        set $node "";      # 后端主机地址
        set $proxy_url ""; # 后端页面地址
        set $proxy_scheme http; # 后端协议，默认 http
        set $proxy_scheme_url ""; # 改造后的代理 path，这里会带上协议约定，如 https-
        # 通过正则提取约定协议、后端节点和后端节点 url
        if ( $request_uri ~* "^/(https-|)([^/]+)/(.*)$" ) {
            set $proxy_scheme_url "$1";
            set $node "$2";
            set $proxy_url "$3";
        }
        # 当协议变量值是 https-的时候，设置代理后端协议为 https，此规则就兼容了后端 https 的情况
        if ( $proxy_scheme_url = "https-" ) {
            set $proxy_scheme https;
        }
        # 不带后端地址直接访问代理 IP 的时候，定向到/html 路径，里面可以放 index.html 导航页面，方便用户点击访问
        location ~ ^/($|static|favicon.ico) {
            break;
        }
        # 当访问的路径没有命中上述规则，且存在字符串的时候，将会进入到这个 location 开始反向代理
        location ~ ^/(.+)/ {
            # 使用 subs_filter 模块进行替换（不了解的话，请自行百度这个关键词）
            # ======================== replace Url begin ========================
            # 替换支持所有类型
            subs_filter_types *;
            # 替换 html 里面的静态资源引用，即 src=、href=等引用形式，另外还考虑到可能存在 / 打头的相对路径或“http://节点”的绝对路径引用形式。
            subs_filter "(href|src)(\s*)=(\s*)('|\")($proxy_scheme://$node|)/" '$1="http://$host/$proxy_scheme_url$node/' igr;
# 替换 js 里面的一些 ajax 请求地址：
            subs_filter "url:(\s*)('|\")/([^'\"]*)('|\")" "url: 'http://$host/$proxy_scheme_url$node/$3'" igr;
 
            # 这里替换实际访问发现还有问题的路径（这里主要是用了 xmlhttprequest 导致上述正则没命中）
            subs_filter "/workerProxy\?ip=" "http://$host/$proxy_scheme_url$node/workerProxy?ip=" igr;
            # you can write replace rule with subs_filter if found more.
            # ======================== replace Url end ===========================
            # 代理到动态后端
            proxy_pass $proxy_scheme://$node/$proxy_url;
            # 关闭 gzip，以防替换不了内容（不能解决后端强制了 gip 压缩的情况...）
            proxy_set_header Accept-Encoding "";
            # 其他 proxy 参数略.
        }
}
```

Tips：实际调试过程中，可以给 Nginx 集成一个 echo 模块，可以将变量打印出来，方便调试。

最后，在 Nginx 服务器的/html 目录放一个 index.html 导航页面，用户只需要访问代理地址 http://myproxy.oa.com/，就能看到全部的后端节点，点击访问即可，真的不要太爽哦！

