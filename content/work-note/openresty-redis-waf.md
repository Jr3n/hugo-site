---
title: Openresty redis WAF
date: 2020-01-03T21:40:09+08:00
draft: false
tags: ["nginx", "WAF", "openresty"]
series: ["Nginx-notes"]
categories: ["0x01工作筆記"]
---
[TOC]
## Lua code
```lua
local redis = require "resty.redis"
local cjson = require("cjson.safe")

local uri = ngx.var.request_uri
local url = ngx.var.host .. uri
local domain = ngx.var.host
local ccCount = tonumber(ngx.var.ccCount) or 100
local ccSeconds = tonumber(ngx.var.ccSeconds) or 10
local redisIp = '127.0.0.1' 
local redisPort = 6379
local blackSeconds = tonumber(ngx.var.blackSeconds) or 300
local geoip = ngx.var.whiteip
local red = redis:new()

local function getClientIp()
        local IP = ngx.req.get_headers()["X-RC-IP"]
        if IP == nil then
                IP = ngx.var.remote_addr 
        end
        if IP == nil then
                IP = "unknown"
        end
        return IP
end

-- get client's cookie to specific the session
local function getClientCookie()
        local COOKIE = ngx.var.cookie_JSESSIONID
        if COOKIE == nil then
                COOKIE = ""
        end
        COOKIE = string.gsub(COOKIE, "JSESSIONID=(%w+)", "%1")
        return COOKIE
end

local function RedKeepAlive()
        -- put it into the connection pool of size 100,
        -- with 10 seconds max idle time
        -- and turn on the lua_code_cache on !!
        local ok, err = red:set_keepalive(10000, 100)
        if not ok then
                ngx.log(ngx.CRIT, "failed to set keepalive: ", err)
                return
        end
end

local function Block()
        local xip = getClientIp()
        local token = xip .. "." .. ngx.md5(domain .. uri .. getClientCookie())
        local req = red:exists(token)

        if req == 0 then
                red:incr(token)
        else
                local times = tonumber(red:get(token))
                if times >= ccCount then
                        local blackReq = red:exists("black." .. token)
                        if (blackReq == 0) then
                                red:set("black." .. token,1)
                                red:expire("black." .. token,blackSeconds)
                                red:expire(token,blackSeconds)
                                RedKeepAlive()
                                ngx.exec("@uriforbid")
                        else
                                RedKeepAlive()
                                ngx.exec("@uriforbid")
                        end
                        return
                else
                        red:incr(token)
                end
        end
        -- check the key's ttl, if the key doesn't have ttl, set ttl to the key.
        local redTtl = red:ttl(token)
        if redTtl == -1 then
                red:expire(token,ccSeconds)
        end
end

red:set_timeout(1000) 
-- Test redis connection
local ok, err = red:connect(redisIp, redisPort)
-- 記錄reused time debug 用
-- local times, err = red:get_reused_times()
-- ngx.log(ngx.CRIT, times)
if not ok then
        ngx.log(ngx.CRIT, "failed to connect: ", err)
        return
end

if (geoip == "1000") then
        Block()
end
RedKeepAlive()
```

### Bug fix
#### 起因
> 因為有部但的客戶會異常的觸發 trigger，被導到限制流量的頁面。
> 後來查看到原因是有部份 key 的 ttl 時間為 -1 (不會 timeout)。
> 進行 code review 後，發現一個地方造成時間差的問題。

```lua
local function Block()
        local xip = getClientIp()
        local token = xip .. "." .. ngx.md5(domain .. uri .. getClientCookie())
        local req = red:exists(token)

        if req == 0 then
                red:incr(token)
                red:expire(token,ccSeconds) -- set ttl to the key
        else
                local times = tonumber(red:get(token))
                if times >= ccCount then
                        local blackReq = red:exists("black." .. token)
                        if (blackReq == 0) then
                                red:set("black." .. token,1)
                                red:expire("black." .. token,blackSeconds)
                                red:expire(token,blackSeconds)
                                RedKeepAlive()
                                ngx.exec("@uriforbid")
                        else
                                RedKeepAlive()
                                ngx.exec("@uriforbid")
                        end
                        return
                else
                        -- if the key timeout before this incr, lua set increase the key value, 
                        -- but doesn't set ttl to the key. So, few keys doesn't have ttl.
                        red:incr(token)
                end
        end
end
```

後來改成在incr後，再進行 ttl 的檢查，如果沒有 ttl 則加上 ttl 的時間。

## Reference
- [基于openresty的后端应用健康检查-动态上下线](https://leokongwq.github.io/2018/01/31/openresty-health-check-dynamic-up-down.html)
- [lua-resty-upstream-healthcheck](https://github.com/openresty/lua-resty-upstream-healthcheck)