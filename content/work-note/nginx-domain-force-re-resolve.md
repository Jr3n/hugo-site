---
title: Setting domains in variables for forcing re-resolution of the DNS names
date: 2020-03-06T18:40:09Z+08:00
draft: true
tags: ["domain", "resolver", "nginx"]
series: 
categories:
toc: true
---
## 起因
> 原先想設定 nginx 進行 proxy_pass 時，能夠依照 domain 當下的解析，進行動態的 proxy_pass。
> 但 nginx 只會在讀取設定的時候才會解析，並將那次的解析做為 proxy_pass 的對象，之後就不會再進行解析。
> 解析有更改的時候，不會跟著一起改變。

## 處理方式
將 proxy_pass 的 domain 存為變數，也因為是變數， Nginx 會動態的去解析 domain！

**Warning！** 在進行解析時，Nginx 不會使用 `/etc/hosts` 進行本地解析，在設定 `resolver` 時，一定要能訪問到該台 DNS server！

在 1.1.9 版之後的NGINX，可以針對 resolver 的 cache TTL 進行調整。

## 設定
```nginx
server {
    ...
    resolver 8.8.8.8;
    set $backend backend.com;
    proxy_pass http://$backend;
    ...
}
```

## Reference
- [How to force nginx to resolve DNS (of a dynamic hostname) everytime when doing proxy_pass?](https://serverfault.com/questions/240476/how-to-force-nginx-to-resolve-dns-of-a-dynamic-hostname-everytime-when-doing-p/593003#593003)