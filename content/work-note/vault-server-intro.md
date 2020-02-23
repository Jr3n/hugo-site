---
title: Vault Intro
date: 2020-02-18T16:40:09Z
draft: true
tags: ["vault"]
series: 
categories:
toc: true
---
## Introduction
**機密資料**，像是 tokens、passwords、credentials 等等。
應用程式往往都會存取這些資料。例如：與第三方API介接需要公私鑰的配對驗證，購物網站的應用程式需要帳號密碼才能連線到資料庫。
對資訊業來說，這些資料的管理，是**非常重要**的一環呢！

大部份的解決方法是將這些資訊存放到某個檔案裡，並在有需要使用的時候讀取。
但是這種方法存在著不少問題：有權限讀取檔案的人，可以取得這些資料，也就是說，可以**取得跟應用程式擁有相同的權限**，而應用程式的權限通常對資料庫具有讀寫的能力！更別說是把資料上傳到`github`等等的公開SCM軟體上，更是增加了不少的危險！

而隨著技術的進步，現今的**應用程式**和**雲環境**可說是更複雜，分散式架構、API key、access token等等…
這些資料如果沒有放好，就會延伸出更多的安全性問題！
而時間到了之後，要一直定期更改這些機密資料，更是頭痛。

由 HachiCorp 所開發的開源軟體 Vault ，可以**解決你我的煩惱**！
這個玩具有幾個特點：
1. 把機密資料`加密`後，再儲存到 database。
2. 可以動態生成有權限的機密資料，並帶有`存活時間(TTL)`、及`註銷(revoke)`的功能。
3. 記錄 client 的蹤跡`(audit)`，留下來存取過的 client 的資料，以利查詢，並加以追蹤。

好啦，說完這麼多，我們就來安裝看看吧！
## Installation
要使用 Vault 之前，當然就是要先下載 Vault 啦，官網很貼心的提供了各平台的 binary package，[官方下載頁](https://www.vaultproject.io/downloads/)。
如果不想使用官方編譯完成的執行檔，也可以從官方提供的 source 自行編譯，[編譯說明文件](https://www.vaultproject.io/docs/install/index.html)

### 驗證安裝
下載完執行檔之後，放到環境變數 `PATH` 的目錄下，才能直接在 terminal 執行。
也請記得打開 terminal ，執行 `vault` ，如果可以看到以下畫面的話，就表示安裝完成啦！
```
$ vault

Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    agent       Start a Vault agent
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit          Interact with audit devices
    auth           Interact with auth methods
    debug          Runs the debug command
    kv             Interact with Vault's Key-Value storage
    lease          Interact with leases
    namespace      Interact with namespaces
    operator       Perform operator-specific tasks
    path-help      Retrieve API help for paths
    plugin         Interact with Vault plugins and catalog
    policy         Interact with policies
    print          Prints runtime configurations
    secrets        Interact with secrets engines
    ssh            Initiate an SSH session
    token          Interact with tokens
```

接下來就能正式開始使用這個神兵利器了呢！

### 同場加映
Vault 提供了命令自動補齊的功能，可以補齊 _subdomain_、_flags_、_path_，非常好用！
但此項功能目前只支援`Bash`、`ZSH`以及`Fish`，其餘 shell 尚不支援。

安裝方法如下：
```bash
$ vault -autocomplete-install
```

執行完後會在 `~/.bashrc` 、 `~/.zshrc` 或是 `~/.config/fish/completions/vault.fish` 加上補齊的相關設定。
之後重新開啟 terminal 就能使用 `vault <tab>` 來進行後續的操作啦。

操作方法就放到下一篇再進行介紹。

## Reference
- [Hashicorp Vault：安全性與複雜性的平衡](https://kknews.cc/zh-tw/tech/9y86or5.html)
- [官方教學](https://learn.hashicorp.com/vault)
- [Hashicorp Vault介绍和使用说明](https://blog.csdn.net/peterwanghao/article/details/83181932)