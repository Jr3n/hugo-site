---
title: Ngrok installation
date: 2019-12-31T16:40:09+08:00
draft: false
tags: ["ngrok"]
series: ["天下武功百百種"]
categories: ["0x01工作筆記"]
---
[TOC]
# Ngrok 安裝
## 起因
> 由於 RD 開發需要讓第三方打回自己電腦上的機器，故使用 Ngrok 處理這項專案。
> Ngrok 看起來如下圖，透過 client 的程式打了一個洞，直接讓第三方進來。
> ![Ngrok](https://process.filestackapi.com/cache=expiry:max/resize=width:700/compress/XoNCgrwnQhOoQMfy1O5T)  
> 圖片來源：[codementor](https://process.filestackapi.com/cache=expiry:max/resize=width:700/compress/XoNCgrwnQhOoQMfy1O5T)
> ### What is ngrok?
> ngrok is a reverse proxy that creates a secure tunnel from a public endpoint to a locally running web service. ngrok captures and analyzes all traffic over the tunnel for later inspection and replay.

## 使用 Google Cloud Platform 的機器實做
> 情境設定：  
> 機器IP: 1.1.1.1  
> Domain: example.com  
### 域名解析
1. 添加一筆 A 記錄 ngrok.example.com 解析到機器 IP 1.1.1.1 上。
2. 添加一筆 CNAME 記錄 *.ngrok.example.com 解析到 ngrok.example.com。
---
### 安裝 go 環境 
#### 從官網下載 tarball
1. 下載 go 的 tar 檔
```bash
wget https://dl.google.com/go/go1.11.4.linux-amd64.tar.gz
```
2. 解壓縮 
```bash
tar -zxvf go1.11.4.linux-amd64.tar.gz -C /usr/local/
```
3. 設定環境變數 /etc/profile.d/go.sh
```bash
export GOROOT="/usr/local/go"
export GOPATH="/usr/local/ngrok"
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
4. 設定完成後執行
```bash
source /etc/profile.d/go.sh
```
#### 使用 yum 安裝
1. 執行 yum install -y go
2. 設定環境變數 /etc/profile.d/go.sh
```bash
export GOROOT="/usr/lib/golang"
export GOPATH="/usr/local/ngrok"
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```
3. 設定完成後執行 `source /etc/profile.d/go.sh`
---
### 從 git 下載 ngrok 的 source code && 生成證書
1. `cd /usr/local/ &&  git clone https://github.com/inconshreveable/ngrok.git` 
2. 執行以下命令，生成新的證書，並取代 source code 的證書。
```bash
export NGROK_DOMAIN="ngrok.example.com"
openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out device.key 2048
openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
cp rootCA.pem $GOPATH/assets/client/tls/ngrokroot.crt
cp device.crt $GOPATH/assets/server/tls/snakeoil.crt 
cp device.key $GOPATH/assets/server/tls/snakeoil.key
```
#### 編譯 server 端的程式
1. 在 /usr/local/ngrok/ 目錄下執行
```bash
make release-server
```
2. 如果一切正常，在 /usr/local/ngrok/bin/ 會有 `ngrokd` 的執行檔。執行命令如下。
```bash
/usr/local/ngrok/bin/ngrokd -domain="ngrok.example.com" -httpAddr=":9527" -httpsAddr=":49527" -tunnelAddr=*":4443"
```
> - httpAddr：訪問 http 使用的 port \(預設 80\)
> - httpsAddr：訪問 https 使用的 port \(預設 443\)
> - tunnelAddr：ngrok 的連接 port \(預設 4443\)
> - 以上的設定都可以改，沒改的話，會使用預設值。\(要檢查一下 iptables 的設定。\)
#### 編譯 client 端的程式
1. 根據 client 的 OS 及位元，export 不同的變數。
   1. 根據 OS：
      - 如果是 linux：`export GOOS=linux`
      - 如果是 windows：`export GOOS=windows`
      - 如果是 MAC：`ecport GOOS=darwin`
   2. 根據位元：
      - 如果是 32-bits 系統：`export GOARCH=386`
      - 如果是 64-bits 系統：`export GOARCH=amd64`

2. 在 /usr/local/ngrok/ 目錄下執行
```bash
make release-client
```
3. 如果一切正常\(此處是生成 windows 64-bits\)，在 /usr/local/ngrok/bin/windows_amd64/ 會有 ngrok.exe 的執行檔，將它 copy 出來。
4. 建立設定檔：ngrok.cfg
```conf
server_addr: "ngrok.example.com:4443"
trust_host_root_certs: false #因為是用自己建的證書編譯出來的，要設定為 false。
```
5. 執行命令 
```bat
ngrok.exe -subdomain=renato -config="ngrok.cfg" 80
```
> - subdomain：設定子域名。
> - config：為上述設定的 config 檔。
> - port：設定要 mapping 出去的 port。\(上為 80\)

啟動後如下：  
![ngrok_client_start_success](https://i.imgur.com/5KHciPV.png)

之後就能用 `http://renato.ngrok.example.com:9527` 進行連線。\(也就是將 `renato.ngrok.example.com:9527` 映射到本地的 `127.0.0.1:80` \)  
![ngrok_connect](https://i.imgur.com/sisw1UU.png)

---
### 利用 Nginx 進行 proxy_pass
如遇到 server 80 port 已被佔用，可以設定 pass 讓資料可以正常的 forward。
```nginx
server {
        listen  80;
        server_name *.ngrok.example.com;

        location / {
                proxy_set_header Host $http_host:9527; # 這個是重點，要設定這個 ngrok 才會正常的 pass 給映射的本機。
                proxy_pass http://127.0.0.1:9527;
        }
}
```

## Reference
- https://yangbingdong.com/2017/self-hosted-build-ngrok-server/
- https://morongs.github.io/2016/12/28/dajian-ngrok/
- https://www.jianshu.com/p/cd937631a88b