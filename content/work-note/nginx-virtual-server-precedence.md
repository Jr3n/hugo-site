---
title: Nginx Virtual Server Precedence
date: 2019-12-31T16:40:09Z
draft: false
tags: ["nginx", "server-name", "listen"]
series: ["Nginx-notes"]
categories: ["0x01工作筆記"]
---
[TOC]
# Nginx Virtual Server Precedence
## Parsing the "listen" Directive to Find Possible Matches
Nginx 會根據 server block 中， `listen` 的設定，去決定哪個 block 回應請求。
### Nginx 會自動補上不足的 listen 設定。
1. 如果沒有 `listen` 的設定， Nginx 會預設給予 `0.0.0.0:80` (or `0.0.0.0:8080` if Nginx is being run by a normal, non-root user)，也就是說每個介面都會 listen 80 port。
2. 如果只給了 IP 會自動補上 80 port，變成 `IP:80`。
3. 如果只給了 port 會自動補上 IP 0.0.0.0 ，變成 `0.0.0.0:port`。

### 由 listen 決定 request 會由哪個 server block 回應
- 補上設定後，Nginx 會找出 request 最 specific 的匹配到哪一個 server block 的 `listen`。
- 如果有特定 IP 已經被設定上去，0.0.0.0 的那個 block 不會被選上。在一般情況下 port 的設定會比較容易被選中。
- 如果在此一階段會 Match 到多個 server block。Nginx 才會開始匹配 `server_name` 的設定。

---
### Example
假設 Server 有兩個IP， 10.1.1.11 跟 10.1.1.12。設定如下：
```nginx
server {
        listen 80; # 補上設定後： 0.0.0.0:80
        server_name test.com;
        charset utf-8;

        location / {
                return 500;
        }
}
server {
        listen 10.1.1.11; # 補上設定後： 10.1.1.11:80
        charset utf-8;

        location / {
                return 405;
        }
}
```
- 假設 *test.com* 解析到 10.1.1.11，此一 Nginx 收到 Host: 標頭為 *test.com* 的 request 的時候，Nginx 會返回 405，而不是500。
  - 因為在 IP 跟 port 的判斷，就能處理了，不用判斷到 `server_name` 。
- 如 Server 只有一個 IP 的話，則會返回 500。
  - 因為對 Nginx 來說，收到的 request 沒辦法找到最 specific 的 block，故會以 `server_name` 去判斷。

---
## Parsing the "server_name" Directive to Choose a Match
如果以 `listen` 沒辦法決定， Nginx 會開始匹配 `server_name` 的設定(對應到 request 的 `Host:` Header)。
### server_name 判斷順序
1. **完全 Match 的 `server_name`：** 會最先被使用。如果有多個完全 Match 的 block，則會使用 **第一個** Match 的 block。
2. **Leading wildcard：** 使用 `server_name` **開頭**為 \* 的 block。如果有多個 Match，則使用 **longest** Match 的那個 block。
3. **Trailing wildcard：** 使用 `server_name` **結尾**為 \* 的 block。如果有多個 Match，則使用 **longest** Match 的那個 block。
4. **Regular expression：** 使用 `~` 開頭(正則式匹配)，Match 後，使用 **第一個** Match 的 block。
5. **Default Server Block：** 如以上皆沒 Match，使用 default server block。
   - Default Server Block 被定義在每個 IP address/port combo 裡。每個 IP address/port combo **第一個** block 就是 Default Server Block。
   - 或是在 `listen` 使用 `default_server` 的參數。(此參數在每個 IP address/port combo，只能有一個)

---
## Reference
- [Digital Ocean](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms "Digital Ocean")