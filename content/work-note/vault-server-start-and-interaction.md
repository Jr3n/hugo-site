---
title: Vault start
date: 2020-02-18T16:40:09Z
draft: true
tags: ["vault"]
series: 
categories:
toc: true
---
# Vault
## Starting Vault Server
安裝完畢後，可以啟動 Vault server 來玩了！

使用以下指令，即可啟動 Vault server：
```bash
$ vault server -dev
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.3.2

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: LLuGlaFVgmqnt40ubuseHYvNvJFalLnbO12q4gBkeAA=
Root Token: s.lazjRdaqKjDNUMUTgmwponKe

Development mode should NOT be used in production installations!

==> Vault server started! Log data will stream in below:
```

### **Warning！** 以上為 dev server ，不適合用在生產環境上！
dev server 特性：
- 資料存在 memory 。
- localhost without TLS。
- automatically unseals and shows you the unseal key and root access key

在啟動完畢後，還要先做幾件事才能連讓 vault client(CLI) 連上 vault server ！
1. 打開 terminal，輸入以下指令：
```bash
$ export VAULT_ADDR='http://127.0.0.1:8200'
# 如果沒有 VAULT_ADDR 這個環境變數的話，預設為 https://127.0.0.1:8200
```
2. 把 **Unseal Key** 記錄下來。
3. 把 **Root Token** 記錄下來，並設為環境變數 `VAULT_DEV_ROOT_TOKEN_ID`
```bash
$ export VAULT_DEV_ROOT_TOKEN_ID="s.lazjRdaqKjDNUMUTgmwponKe"
```

### 驗證 vault server 有正常啟動！
執行 `vault status` 這個指令，如果成功的話，應該會有下面的畫面：

```bash
$ vault status

Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.3.2
Cluster Name    vault-cluster-d871077f
Cluster ID      fb0e5b60-bef0-bff9-295d-146f19af2dc7
HA Enabled      false
```

接下來就可以跟 vault server 進行互動了！

---

## Interaction with Vault Server
Vault 的 CLI 是透過 vault's HTTP API 去跟 Vault server 進行互動。
我們寫進 Vault 的資料，會先進行加密，然後存進 backend storage，backend storage 不會知道所存的資料為何。
以 dev mode server 來說，是存到 memory 裡的，在生產環境的話，可以參考[官方文件]([官方文件](https://www.vaultproject.io/docs/configuration/storage/))存到硬碟或是[Consul](https://www.consul.io/) 裡。

### Writing a secret
透過以下指令，把 `author=renato` 以及 `topic=vault` 存到 `secret/7days` 這個路徑底下。
```bash
$ vault kv put secret/7days author=renato topic=vault

Key              Value
---              -----
created_time     2020-02-19T06:38:06.985724259Z
deletion_time    n/a
destroyed        false
version          1
```

**Warning！** 如果要存放 key-value 的話，從 file 讀取的話會更加安全，透過指令的方式存放的話，會被記錄在 `history` 裡面。

### Getting a secret
透過以下指令，讀取 `secret/7days` 的內容，可以加上參數去顯示要的範圍及格式。
例如：`-field=renato`、`-format=json`。
```bash
$ vault kv get secret/7days 

====== Metadata ======
Key              Value
---              -----
created_time     2020-02-19T06:41:37.647023263Z
deletion_time    n/a
destroyed        false
version          1

===== Data =====
Key       Value
---       -----
author    renato
topic     vault
```

### Deleting a secret
```bash
$ vault kv delete secret/7days

Success! Data deleted (if it existed) at: secret/7days
```