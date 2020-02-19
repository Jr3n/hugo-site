---
title: Vault Intro
date: 2020-02-18T16:40:09Z
draft: true
tags: ["vault"]
series: 
categories:
toc: true
---
# Vault
## Introduction
機密資料管理，在資訊業可說是非常重要的一環，像是 tokens、passwords、credentials 等等。
應用程式往往都會存取這些資料。例如：購物網站的應用程式需要帳號密碼才能連線到資料庫、與第三方支付需要公私鑰的配對驗證。

簡單的解決方案是將這些資訊儲存在配置檔案中，並在啟動時讀取它們。但是這種方法的問題顯而易見：有權訪問此檔案的人共享我們的應用程式具有的資料庫許可權 - 通常可以完全訪問所有儲存的資料。

我們可以嘗試通過加密這些檔案來使事情變得更加困難。但是，這種方法在整體安全性方面不會增加太多。主要是因為我們的應用程式必須能夠訪問主金鑰。當以這種方式使用時，加密僅是一種“錯誤”的安全感。

現代應用程式和雲環境往往會增加一些額外的複雜性：分散式服務，多個數據庫，訊息傳遞系統等等，所有敏感資訊都在各處傳播，從而增加了安全漏洞的風險。

所以，我們能做些什麼？來用下Vault吧！
由 HachiCorp 所開發的開源軟體 Vault ，

## Installation
要使用 Vault 之前，當然就是要先下載 Vault 啦，官網很貼心的提供了各平台的 binary package，[官方下載頁](https://www.vaultproject.io/downloads/)。
如果不想使用官方編譯完成的執行檔，也可以從官方提供的 source 自行編譯，[編譯說明文件](https://www.vaultproject.io/docs/install/index.html)

### 驗證安裝
下載完執行檔之後，放到環境變數 `PATH` 的目錄下，才能直接在 terminal 執行。
也請記得打開 terminal ，執行 `vault` ，如果可以看到 help 的畫面的話，就表示安裝完成啦！可以正式開始使用這個神兵利器了！

### 同場加映
Vault 提供了命令自動補齊的功能，可以補齊 _subdomain_、_flags_、_path_，非常好用！
但此項功能目前只支援`Bash`、`ZSH`以及`Fish`，其餘 shell 尚不支援。

安裝方法如下：
```bash
$ vault -autocomplete-install
```

執行完後會在 `~/.bashrc` 、 `~/.zshrc` 或是 `~/.config/fish/completions/vault.fish` 加上補齊的相關設定。
之後重新開啟 terminal 就能使用 `vault <tab>` 來進行後續的操作啦。