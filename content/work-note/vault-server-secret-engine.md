---
title: Vault secrets engine - AWS
date: 2020-02-18T16:40:09Z
draft: true
tags: ["vault"]
series: 
categories:
toc: true
---
# Vault
## Secrets engine
簡單操作完 key-value 的存放後，看官們是不是會好奇，為什麼所操作的指令都帶有 `secret/`，如果反骨一點，要在 `2/14` 這個路徑下存放 `iHaveAGirlfreind=true` 呢？

在剛啟動的 vault server 進行以下操作，只會得到相對應的~~錯誤~~ 403(哭哭)。
```bash
$ vault kv put 2/14 iHaveAGirlfreind=true

Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/internal/ui/mounts/2/14
Code: 403. Errors:

* preflight capability check returned 403, please ensure client's policies grant access to path "2/14/"
```

之所以前面的操作使用 `secret/`，是因為 Vault server 在預設啟動的時候，就會在 `secret/` 這個路徑下，啟動 `kv` 類型的 secrets engine。

而路徑是提供 Vault server 知道要去哪個 secrets engine 做存放，像是 filesystem 的存放。

我們可以透過跟不同的進行互動 secrets engine ，存放不同類型的資料。

詳細各類型的 secrets engine 的使用方式，可以到[官方網站](https://www.vaultproject.io/docs/secrets/)查看。

接下來我們就來操作一下 secrets engine 的使用！

### Enable a secret engine
進行 secret engine 的啟用，預設就會增加一個 path 出來，每個 path 之間是獨立且不能互相訪問的。
執行以下的指令，可以在 `aws/` 的路徑下，建立一個 `aws` secrets engine。 

```bash
$ vault secrets enable -path=aws aws
# 如未帶 path 參數，會直接以 secrets engine 的名字建立。
$ vault secrets enable aws
```

建立完成後，可以執行 `vault secrets list` 進行確認，並看到 enabled secrets engine 的細詳內容。

```bash
$ vault secrets list

Path           Type         Accessor              Description
----           ----         --------              -----------
aws/           aws          aws_16428af2          n/a #沒有設定 Description 的話，會顯示 n/a
cubbyhole/     cubbyhole    cubbyhole_dc16024b    per-token private secret storage
identity/      identity     identity_394049ce     identity store
secret/        kv           kv_9397ee72           key/value secret storage
sys/           system       system_fd0af39a       system endpoints used for control, policy and debugging
```

> `sys/` 這個路徑跟 vault 的核心設定有關，不是 secrets engine 的一種！

### Disable a secret engine
已經不需要 secrets engine 的話，可以直接刪除。
當 secrets engine 被刪除的時候，裡面所有的資料以及設定也會一併被刪除！

```bash
$ vault secrets disable aws/

Success! Disabled the secrets engine (if it existed) at: aws/
```

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

#### Configure the AWS secrets engine
```bash
$ vault write aws/config/root \
    access_key=<access_key> \
    secret_key=<secret_key> \
    region=us-east-1

Success! Data written to: aws/config/root
```

#### Create a role
接下來建立 role，讓 vault 在 AWS 上建立 IAM users 時，提供相對應的權限。
相關的 policy 可以在 [AWS官網](https://awspolicygen.s3.amazonaws.com/policygen.html) 上查詢。
```bash
$ vault write aws/roles/my-role \
         credential_type=iam_user \
         policy_document=-<<EOF
 {
   "Version": "2012-10-17",
   "Statement": [
     {
       "Effect": "Allow",
       "Action": [
         "ec2:DescribeInstances",
         "iam:GetInstanceProfile",
         "iam:GetUser",
         "iam:GetRole"
       ],
       "Resource": "*"
     }
   ]
 }
EOF
Success! Data written to: aws/roles/my-role
```

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