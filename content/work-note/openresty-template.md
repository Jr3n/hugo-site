---
title: Openresty template
date: 2020-01-03T21:40:09Z
draft: true
tags: ["nginx", "template", "openresty"]
---
[TOC]
## 起因
> 當機房的網路設備要維護時，原本統一回傳簡單的靜態頁面。
> 希望能依據 domain 回傳不同的內容給使用者。 
> 但官方的文件的資源不多，後來找了非官方的`lua-resty-template`使用，並搭配 redis ，使頁面可以動態的調整。

## Installation
### Git clone 
```bash
$ git clone https://github.com/bungle/lua-resty-template.git
```
把 `template.lua` 跟 `template` directory 放到 library 的路徑下，並要在 `nginx.conf` 的 `lua_package_path` 加上路徑。

### Using OpenResty Package Manager (opm)
```bash
$ opm get bungle/lua-resty-template
```

## Nginx configuration

### Template path
要設定 template 的路徑，讓 nginx 去找檔案。
如果沒有設定的話，會以 `root`(`ngx.var.document_root`) 當路徑。

```nginx
set $template_root /usr/local/nginx/conf/templates;
set $template_location /templates;
```

- path precedence：
  1. `template_location` (如果 status code 非 200，會往下找)
  2. `template_root`
  3. `ngx.var.document_root`

---

### 搭配 redis
在 nginx access 階段，使用域名當 key ，去 redis 撈取 value：

#### key-value 如下：
1.  Key: `example.com`
    Value:
    ```json
    '{
        "siteId":"EX123",
        "domainType":"Domain",
        "siteType":"VIP"
    }'
    ```
2.  Key: `VIP`
    Value:
    ```json
    '{
        "isMaintain":"False"
    }'
    ```
3.  Key: `EX123`
    Value: 
    ```json
    '{
        "startTime": "1997-03-27 03:00:00",
        "endTime":"1997-03-27 06:00:00",
        "serviceUrl":"https://www.google.com"
    }'
    ```

#### access.lua
```lua
local redis = require "resty.redis"
local cjson = require("cjson.safe")

local domain = ngx.var.host
local redisIp = '127.0.0.1'
local redisPort = tonumber(ngx.var.redisport) or 6379
local red = redis:new()
red:set_timeout(1000)
-- Test redis connection
local ok, err = red:connect(redisIp, redisPort)
-- record reused time, for debug
-- local times, err = red:get_reused_times()
-- ngx.log(ngx.CRIT, times)
if not ok then
        ngx.log(ngx.CRIT, "failed to connect: ", err)
        return
end

-- get the value by using domain as key
local domainData, err = red:get(domain)
if not domainData then
        ngx.log(ngx.CRIT, "failed to get ", domain .. ": " .. err)
        return
end

local domainJson = cjson.decode(domainData)
-- get maintain data by using siteId as key
if domainJson ~= nil then
        local siteData, err = red:get(domainJson.siteType)
        if not siteData then
                ngx.log(ngx.CRIT, "failed to get ", domainJson.siteType .. ": " .. err)
                return
        end

        local siteJson = cjson.decode(siteData)
        if siteJson ~= nil then
                if siteJson.isMaintain == "True" then
                        local res, err = red:get(domainJson.siteId)
                        if not res then
                                ngx.log(ngx.CRIT, "failed to get ", domainJson.siteId .. ": " .. err)
                                return
                        end

                        if res == ngx.null then
                                RedKeepAlive()
                                -- redirect to location @maintain
                                ngx.exec("@maintain")
                        end

                        if res ~= ngx.null then
                                RedKeepAlive()
                                -- set res to nginx var 
                                -- ngx.var.res is nginx's variable
                                -- res is lua's variable
                                ngx.var.res = res
                                ngx.var.domainType = domainJson.domainType
                                ngx.exec("@maintain")
                        end
                end
        end
end
```

當 `isMaintain` 為 `True` 的時候，就會導進維護頁。並把拿到的 value 塞到 nginx 的變數 res，好讓 content 階段可以使用。

---

### 使用 template 處理 response
在 nginx content 階段，使用 template 處理 response。
```nginx
location @maintain {
    default_type 'text/html';
    content_by_lua_file /path/to/nginx/lua/maintain.lua;
}
```

#### maintain.lua
```lua
local template = require("template")  --import template module
local cjson = require("cjson.safe")

-- using nginx var as data
local res = ngx.var.res
local domainType = ngx.var.domainType
local sTime, eTime
-- If site is null return nomatch html
if res == ngx.null then
        template.render("nomatch.html")
        return
else
        -- If res's value can not decode by json
        local resJson = cjson.decode(res)
        if resJson == nil then
                template.render("nomatch.html")
                return
        end

        if domainType == ""
                sTime = resJson.startTime
                eTime = resJson.endTime
        else
                template.render("nomatch.html")
                return
        end

        local content = {
                startTime = sTime,
                endTime = eTime,
                serviceUrl = resJson.serviceUrl
        }
        template.render("index.html", content)
        return
end
```

#### template html
```html
{(header.html)}
<body>
    <div id="pic" class="warn_pic">
        <img src="/images/maintain.png" alt="webstie_maintaining" />
    </div>
    <div id="txt" class="warn_txt">
                <h1 id = "ipAddr"></h1>
        <h2>伺服器維護中</h2>
        <p>
            為了提供更好的服務，目前網站正在進行維護<br />
            維護將預計於 {* endTime *} 完成<br />
            {% if serviceUrl ~= '' then %}
            如有任何疑問<br />
            歡迎聯繫我們的<a href={* serviceUrl *}>客服人員</a><br />
            {% end %}
        </p>
    </div>
</body>
</html>
```
{(template)}：include 另一個 template 進來。
{* expression *}：將變數帶進 html 裡。
{% lua code %}：執行 lua 的程式，像是 if, for 等等。

## Reference
- [Lua模版渲染](https://www.kancloud.cn/inwsy/project/1129452)
- [lua-resty-template](https://github.com/bungle/lua-resty-template)