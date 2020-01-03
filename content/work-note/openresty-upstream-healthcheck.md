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
> 但官方的文件的資源不多，後來找了非官方的`lua-resty-template`使用。

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
要設定 template 的路徑，讓 nginx 去找檔案。
如果沒有設定的話，會以 `root`(`ngx.var.document_root`) 當路徑。

```nginx
set $template_root /usr/local/nginx/conf/maintain;
set $template_location /templates;
```
- path precedence：
  1. `template_location` (如果 status code 非 200，會往下找)
  2. `template_root`
  3. `ngx.var.document_root`


```lua
local template = require("template")  --加入 template module
local cjson = require("cjson.safe")

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

        if domainType == "Member" or domainType == "Mobile" then
                sTime = resJson.memberStartTime
                eTime = resJson.memberEndTime
        elseif domainType == "Plat" then
                sTime = resJson.platStartTime
                eTime = resJson.platEndTime
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
## Reference
- [Lua模版渲染](https://www.kancloud.cn/inwsy/project/1129452)
- [lua-resty-template](https://github.com/bungle/lua-resty-template)