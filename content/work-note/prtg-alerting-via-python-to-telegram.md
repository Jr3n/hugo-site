---
title: Send alert from PRTG to Telegram
date: 2020-03-17T16:40:09Z
draft: true
tags: ["python", "telegram", "prtg"]
series: 
categories:
toc: true
---
[TOC]
## 起因
懶癌發作，不想一直盯著螢幕看，所以在PRTG上設定 trigger ，好讓他自行發送通知。
但因為舊版PRTG沒有原生 **發送 telegram 的功能**，使用其 **HTTP Action** 的功能，又會遇到 SSL 的問題，最後找到了使用 python 實作的方法。

## 實作
首先要先申請 TelegramBot ！ 官方網站寫的蠻清楚的，在此就不多加贅述啦！
參考連結： https://core.telegram.org/bots#6-botfather

接下來在 server 上安裝 Python2
安裝完成後，在 PRTG server 安裝的目錄下，建立兩個檔案。
參考路徑：`C:\Program Files (x86)\PRTG Network Monitor\Notifications\EXE`

1. PRTG_telegram.py
```python
import httplib, urllib, sys
 
chat_id = sys.argv[1]
text = sys.argv[2]
msg = ''
 
for count, arg in enumerate(sys.argv):
    if count >= 2:
        msg += arg + '\n'
 
conn = httplib.HTTPSConnection('api.telegram.org',443)
conn.request("POST", "/bot<botToken>/sendMessage",
    urllib.urlencode({
        "chat_id": chat_id,
        "text": msg,
    }),
    { "Content-type": "application/x-www-form-urlencoded" })
response = conn.getresponse()
```

2. PRTG_telegram.bat
```bat
C:\Python27\python.exe "C:\Program Files (x86)\PRTG Network Monitor\Notifications\EXE\PRTG_telegram.py" %*
```

## PRTG setup
### Notifications setting
1. 登入 PRTG
2. 點擊 Setup > Account Settings > Notifications.
3. 點擊底部的 "Add New Notification" 
4. Add a name for your notification – PRTG – ALERT
5. 在 Execute Program > Program file 的地方選擇 `PRTG_telegram.bat`
6. 在 Execute Program > Parameter 寫上：
   ```
   "<chat_id>" "IDC PRTG" "" "Device : %device" "" "Sensor : %name" "" "Status : %status" "" "Downtime : %down" "" "(%message)"
   ```
7. 存檔後，點擊 test 進行測試，收到訊息的話，代表設定成功。

### Trigger setting
在各個 sensors 下面新增 trigger，並套用上剛剛的notification，等 trigger 觸發，就可以收到通知啦！

## Reference
- [PUSH NOTIFICATIONS WITH PRTG AND PUSHOVER](https://myrandomthoughts.co.uk/2015/01/push-notifications-with-prtg-and-pushover/)