---
title: Openresty upstream health check
date: 2020-01-03T21:40:09+08:00
draft: false
tags: ["nginx", "healthcheck", "openresty"]
series: ["Nginx-notes"]
categories: ["0x01工作筆記"]
---
[TOC]
## 起因
> 在發佈期間，想讓 nginx ，判斷後端的狀態，達到自動上下線，不用手動切換。

## nginx 設定
```nginx
http {
    lua_package_path "/path/to/lua-resty-upstream-healthcheck/lib/?.lua;;";

    ## 指定共享內存，可依 upstream 的大小進行調整。
    lua_shared_dict healthcheck 1m;

    ## 範例 upstream block:
    upstream foo {
        server 127.0.0.1:12354;
        server 127.0.0.1:12355;
        server 127.0.0.1:12356 backup;
    }

    upstream bar {
        ...
    }

    ## 在worker初始化時，啟動定時器，進行後端節點的檢查
    init_worker_by_lua_block {
        local hc = require "resty.upstream.healthcheck"
        local ok, err = hc.spawn_checker {
            -- shm 指定共享內存區，
            shm = "healthcheck",
            -- type 指定 healthcheck 的方法，HTTP or TCP，目前只支援 http
            type = "http",
            -- upstream 指定要檢查的 upstream
            upstream = "foo",
            -- 設定 HTTP 請求所發的 request
            http_req = "GET /uri/to/check HTTP/1.0\r\nHost: foo\r\n\r\n",
            -- 請求間隔時間，default 為 1000 ms。最小值為 2 ms。
            interval = 2000,
            -- request timeout 時間。default 為 1000 ms。
            timeout = 1000,
            -- 失敗多少次後，將節點標記為 down。 default 為 5 次。
            fall = 3, 
            -- 成功多少次後，將節點標記為 up。 default 為 2 次。
            rise = 2,
            -- 收到哪些 http status code 為正常。
            valid_statuses = {200, 302},
            -- 同時能發多少次測試， default 為 1。
            concurrency = 1,
        }

        if not ok then
            ngx.log(ngx.ERR, "failed to spawn health checker: ", err)
            return
        end
        ## 有幾個 upstream 要監控，就寫幾次。
        ok, err = hc.spawn_checker{
            shm = "healthcheck",
            upstream = "bar",
            ...
        }

        if not ok then
            ngx.log(ngx.ERR, "failed to spawn health checker: ", err)
            return
        end

        -- Just call hc.spawn_checker() for more times here if you have
        -- more upstream groups to monitor. One call for one upstream group.
        -- They can all share the same shm zone without conflicts but they
        -- need a bigger shm zone for obvious reasons.
    }

    ## 配置 healthcheck result page
    server {
        ...

        # status page for all the peers:
        location = /status {
            access_log off;
            allow 127.0.0.1;
            deny all;

            default_type text/plain;
            content_by_lua_block {
                local hc = require "resty.upstream.healthcheck"
                ngx.say("Nginx Worker PID: ", ngx.worker.pid())
                ngx.print(hc.status_page())
            }
        }
    }
}
```

## healthcheck result page
產生所有 upstream healthcheck 的結果，如果該 upstream 沒有設定 checker，會被標上 `(NO checkers)`。
```
Upstream foo
    Primary Peers
        127.0.0.1:12354 up
        127.0.0.1:12355 DOWN
    Backup Peers
        127.0.0.1:12356 up

Upstream bar
    Primary Peers
        127.0.0.1:12354 up
        127.0.0.1:12355 DOWN
        127.0.0.1:12357 DOWN
    Backup Peers
        127.0.0.1:12356 up

Upstream notchecker (NO checkers)
    Primary Peers
        127.0.0.1:12354 up
        127.0.0.1:12355 up
    Backup Peers
        127.0.0.1:12356 up
```

## Reference
- [基于openresty的后端应用健康检查-动态上下线](https://leokongwq.github.io/2018/01/31/openresty-health-check-dynamic-up-down.html)
- [lua-resty-upstream-healthcheck](https://github.com/openresty/lua-resty-upstream-healthcheck)