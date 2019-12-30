---
title: Nginx Location 處理
tags: nginx, location
notebook: Nginx
---
# Nginx Location 處理
## Directive Syntax Explained 符號說明
### Prefixed match
1. **`(none)`** 
   - No modifier at all means that the location is interpreted as a prefix. To determine a match, the location will now be matched against the beginning of the URI.
   - 最一般的表示法。
2. **`^~`**
   - Assuming this block is the best non-RE match, a carat followed by a tilde modifier means that RE matching will not take place. 
   - 如果 Match 到這個 location 的話，後續的 RE match 不會被處理。
    ```nginx
    location  ^~ /image/test {
        return 406;
    }
    location ~ /image/(.*) {
        return 400;
    }
    ```
   - `$request_uri` 為 /image/test(.\*) 時會回 406，其餘 /image/*other* ，都會回400。
3. **`=`**
   - The equal sign can be used if the location needs to match the exact request URI. When this modifier is matched, the search stops right here.
   - 精確匹配，只要 Match 到這個，就會停止搜尋。
   - Nginx 也真的就是精確匹配。收到的 Request 多一點少一點都不會使用。
    ```nginx
    location / {
        deny all;
    }
    location = /image/ {
        return 405;
    }
    ```
   - 以上述為例，只有 `$request_uri` 為 /image/ 時才會回 405，其餘 /image/*other* ，都會回403。
  
### Regular expressions (RE) match
1. **`~`** 
   - Tilde means that this location will be interpreted as a case-**sensitive** RE match.
   - 正則式，分大小寫。
2. **`~*`** 
   - Tilde followed by an asterisk modifier means that the location will be processed as a case-**insensitive** RE match.
   - 正則式，不分大小寫。

## The Process of Choosing Nginx Location Blocks (Location Match 順序)
1. `location =` *Location*
2. `location ^~` *Location*
3. `location ~,~\*` *Location*
4. `location` *Location*
5. `location` / \(被視為是default設定\)

---
## Reference
- [Digital Ocean](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms "Digital Ocean")
- [Sean's Notes](http://seanlook.com/2015/05/17/nginx-location-rewrite/ "Sean's Notes")