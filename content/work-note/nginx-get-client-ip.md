---
title: Three ways to get client IP on Nginx server
date: 2019-12-31T16:40:09Z
draft: false
tags: ["nginx", "remote-address", "x-forwarded-for"]
series: ["Nginx-notes"]
categories: ["0x01工作筆記"]
---
[TOC]
## 起因
> 由於發現有大量的IP訪問Server，排查後發現，收到的請求插有X-Forwarded-For的標頭。 對方在此一標頭塞入偽造的IP，而 Nginx Waf 所判斷的IP剛好是XFF(如下方程式碼)，故沒辦法有效的擋下請求。
> ```nginx
> function getClientIp()
> IP = ngx.req.get_headers()["X-Real-IP"]
> if IP == nil then
> IP = ngx.req.get_headers()["X_Forwarded_For"]
> end
> if IP == nil then
> IP = ngx.var.remote_addr
> end
> if IP == nil then
> IP = "unknown"
> end
> return IP
> end
> ```

### 處理方式
1. 將程式碼有問題的地方修掉，如下：
   ```nginx
   function getClientIp()
   IP = ngx.req.get_headers()["X-RC-IP"]
   if IP == nil then
   IP = ngx.var.remote_addr
   end
   if IP == nil then
   IP = "unknown"
   end
   return IP
   end
   ```
2. 針對 CDN 過來的請求，原先是沒有擋，要把WAF的功能加上。

## 抓取真實IP
從CDN的請求抓取用戶的IP，有以下常見的三個方式：

### 使用CDN自定義標頭來獲取
請 CDN 廠商加上自定義標頭，如果以上述我司的設定的話，就是請廠商加上 X-RC-IP 這個標頭。
寫成 nginx 的話會長這樣：
```nginx
proxy_set_header X-RC-IP $remote_addr;
```

這樣我司的 nginx proxy 就會收到內容為客戶IP的標頭。 但不是所有的 CDN 廠商都能支援這樣的需求，大多都是加上廠商自行定義好的標頭，此一方法可以接受，但需要再多判別其他的標頭來處理。

```nginx
function getClientIp()
IP = ngx.req.get_headers()["X-RC-IP"]
if IP == nil then
IP = ngx.req.get_headers()["X_CDN_A_IP"]
end
if IP == nil then
IP = ngx.req.get_headers()["X_CDN_B_IP"]
end
if IP == nil then
IP = ngx.var.remote_addr
end
if IP == nil then
IP = "unknown"
end
return IP
end
```

如上方程式碼所寫，如果此一用戶是經由 CDN_B 廠商進來，但帶有 `X_CDN_A_IP` 的標頭，我司的 server 收到後會先以這個當成客戶IP，這樣就形成了誤判。

延伸閱讀：[nginx反向代理proxy_set_header自定义header头无效](http://www.ttlsa.com/nginx/nginx-proxy_set_header/)

### 透過 X_Forwarded_For 來獲取

一般情況下，CDN 廠商的 server 都會傳送 X_Forwarded_For 的標頭過來，此標頭的內容可以輕易偽造，且內容可能為多個 IP。
需要更多的資訊才能進行過濾或是判斷。

延伸閱讀：[HTTP 请求头中的 X-Forwarded-For](https://imququ.com/post/x-forwarded-for-header-in-http.html)

### 用 Nginx 自建模組來獲取

安裝 Nginx 時，加上 realip 的模組。 在編譯時加上 `--with-http_realip_module` 即可啟用。

#### 配置語法
```nginx
set_real_ip_from 192.168.1.0/24; #將此一 IP 設定為可信任IP，以此例來說，就是 CDN 的 server IP。
set_real_ip_from 192.168.2.1;
real_ip_header X-Forwarded-For; #從哪個 Header 搜尋出要的 IP。
real_ip_recursive on; #從最後一個IP開始，去除 set_real_ip_from 所設定的可信任 IP，直到出現不可信任的 IP。
```

#### 情境假設

情境：
- 未配置 realip 模組
  ![](https://raw.githubusercontent.com/alee801223/images/master/Non-real-ip.png)

---

情境：
- 配置 realip 模組
- 且 CDN IP 被加入 set_real_ip_from 名單：
  ![](https://raw.githubusercontent.com/alee801223/images/master/real-ip.png)
- 因 CDN IP 有加入 set_real_ip_from 名單，  `$remote_addr` 被置換成 X_Forwarded_For。

---

情境：
- 配置 realip 模組
- Client 透過自建 proxy 進來
- CDN IP 被加入 set_real_ip_from 名單
  ![](https://raw.githubusercontent.com/alee801223/images/master/real-ip-customer-proxy.png)
- 如客戶是自建 proxy ，且需要過濾掉 CProxy IP 的話，也要將此一 CProxy 的 IP 加入信任名單內。
- 加入後 $remote_addr 會被置換成 Client IP

---

情境：
- 配置 realip 模組
- Client 透過自建 proxy 進來 
- CDN IP 被加入 set_real_ip_from 名單
- Client 加入 偽造 X_Forwarded_For
  ![](https://raw.githubusercontent.com/alee801223/images/master/real-ip-fake-ip.png)

## 三種在CDN環境下獲取用戶IP方法總結

- CDN自定義header
  - Pros：獲取到最真實的用戶 IP，無法偽裝。
  - Cons：需要 CDN 廠商提供，能自定義為最佳解。

- 獲取 x-forwarded-for
  - Pros：可以獲取到用戶的 IP
  - Cons：由於可能為多個 IP，程式需要改動，此外內容有可能是偽造的 

- 使用realip獲取
  - Pros：程式不需要改動，直接使用 $remote_addr 即可獲取 IP
  - Cons：IP 有可能被偽裝，而且需要知道所有 CDN 節點 IP

## Reference
- [运维之美](https://www.hi-linux.com/posts/53006.html "运维之美")
- [維運生存時間](http://www.ttlsa.com/nginx/nginx-get-user-real-ip/ "維運生存時間")