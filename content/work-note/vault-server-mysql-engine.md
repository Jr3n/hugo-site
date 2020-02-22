---
title: Vault secrets engine - Database
date: 2020-02-21T16:40:09Z
draft: true
tags: ["vault"]
series: 
categories:
toc: true
---
# Vault
## Database secrets engine

簡單的玩了一下 AWS secrets engine，接下來來玩一下 database 的 secrets engine 吧！

首先先在 `mysql/` 路徑下建立一個 database secrets engine。
```bash
$ vault secrets enable -path=mysql database
```

之後需要建立一台 mysql 以及 一個有權限的帳號。(當然直接用 root 也是可以的XD)

```sql
-- 在 mysql 執行以下指令。
CREATE DATABASE foo;
CREATE USER 'vault'@'%' IDENTIFIED BY 'vault';
-- 提供 vault 能賦予 user foo DB 的權限。
GRANT ALL PRIVILEGES ON foo.* TO 'vault'@'%' WITH GRANT OPTION;
-- 提供 vault 能夠建立 user 的權限。
GRANT CREATE USER ON *.* to 'vault'@'%';
```

接下來，開始針對 database secrets engine 來做相關設定。

#### 連線資訊
```bash
$ vault write mysql/config/database \
    plugin_name="mysql-database-plugin" \
    connection_url="{{username}}:{{password}}@tcp(mysql-ip:3306)/" \
    allowed_roles="my-role" \
    username="vault" \
    password="vault"
```

#### Create a role
```bash
$ vault write mysql/roles/my-role \
    plugin_name=mysql-database-plugin \
    db_name=my-mysql-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON foo.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```
與 aws 類似，可以做一些設定上的調整！
`creation_statements` 定義的權限，可以自行更改。
`default_ttl` 定義的是預設的 credential 存活時間。
`max_ttl` 定義，如果持續有在使用產生的 credential 的話，credential 的最大存活時間。

### Dynamic secrets engine
跟 `kv` secrets 需要自行存放資料不同，dynamic secrets 只會在讀取的時候產生。
Vault server 產生後的 dynamic secrets 可以藉由 revocation 機制去註銷，因些安全性很高！

上述已建立的 aws secrets engine 就是 dynamic secrets 的一種。

在建立完成後，aws secrets engine 要設定驗證後，才能跟 AWS 進行連線。
要進行驗證，當然需要一組 credentials，此組 credentials 需要有 IAM 管理權限。
可以到 AWS 的 IAM 進行建立一組有 IAM 管理權限的 credentials。

> 如果權限不足的話在執行時會跳出 error，
> Error putting user policy: AccessDenied: XXXXX
> status code: 403, request id: 4372feb6-3c27-435c-90f0-4c9f086a25d8

#### Generate the secret
建立完成後，就能拿到具有 `my-role` 所定義的權限的 access_key 跟 secret_key 啦！

```bash
$ vault read aws/creds/my-role

Key                Value
---                -----
lease_id           aws/creds/my-role/0NNbJwHlU8JkGENsLJyAN2UD
lease_duration     768h
lease_renewable    true
access_key         AKIA6FNTDNMYVKWUJYUB
secret_key         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
security_token     <nil>
```

#### Revoke the secret
```bash
$ vault lease revoke aws/creds/my-role/UgFGPJDeWTwHoIrQhJ31FdAI

All revocation operations queued successfully!
```

## Ref
- [Manage Your MySQL Database Credentials with Vault](https://mysqlrelease.com/2018/05/manage-your-mysql-database-credentials-with-vault/)