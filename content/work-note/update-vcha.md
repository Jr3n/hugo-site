---
title: Upgrade vCenter HA
date: 2020-01-31T17:40:09Z
draft: false
tags: 
---
## 緣由
> zabbix 抓不到部份的監控值，猜測是因為VC 版本問題，故升版看是否能正常收到監控數據。VC 更新時，服務不受影響。
> 下方為說明時示範用IP：
> - vCenter MGT IP： `10.1.0.10`。
> - Active node： `10.1.0.1`。
> - Passive node： `10.1.0.2`。
> - Witness node： `10.1.0.3`。

## 步驟
### 下載安裝 patch
下載要升版的 patch，[官方連結](https://my.vmware.com/group/vmware/patch)
在這邊以 `6.5.0U3f` 做為示範。

![](https://raw.githubusercontent.com/alee801223/images/master/20200114170930.png)

並將檔案傳到 VM host 上。

### 切換 Maintenance Mode
登入到 vSphere Web Client，將 vCenter 切換為 Maintenance Mode。

![](https://raw.githubusercontent.com/alee801223/images/master/20200114171833.png)

### 進行更新
用 ssh 登入到 vCenter(`10.1.0.10`)，登入時，會是登入到 Active Node 的 server 上。
(此時進入shell以指令 `ip a` ，顯示的 IP 會是 `10.1.0.1`)

#### Witness node
登入到 Active node 後，再登入到 Witness node (`10.1.0.3`)上。(IP 會在 HA IP Address 那邊)。
> 登入後，不用切到 _shell_ 裡，在 _appliancesh_ 即可。(如圖)  
> ![](https://raw.githubusercontent.com/alee801223/images/master/20200114172439.png)

接著將要更新的 patch 檔，掛載到 Witness node 上，並輸入以下指令。
`software-packages install --iso --acceptEulas`

執行結果如下：
```
Command> software-packages install --iso --acceptEulas
 [2020-01-14T16:26:35.014] : ISO mounted successfully
 [2020-01-14T16:26:36.014] : Staged 205 packages.
 [2020-01-14T16:26:36.014] : Verifying staging area
 [2020-01-14T16:26:36.014] : ISO unmounted successfully
 [2020-01-14T16:26:36.014] : Validating software update payload
 [2020-01-14T16:26:36.014] : Compatible patch
 [2020-01-14T16:26:36.014] : The version check is passed.
 [2020-01-14T16:26:36.014] : Validation successful
 [2020-01-14 16:26:36,749] : Copying software packages  [2020-01-14T16:26:36.014] : ISO mounted successfully
205/205
 [2020-01-14T16:26:51.014] : ISO unmounted successfully
 [2020-01-14 16:26:51,190] : Running test transaction ....
 [2020-01-14 16:27:04,227] : Running pre-install script.....
 [2020-01-14T16:27:25.014] : All VMware services are stopped.
 [2020-01-14 16:27:25,900] : Upgrading software packages ....
 [2020-01-14 08:31:32,161] : Running post-install script.....
 [2020-01-14T08:32:34.014] : Packages upgraded successfully, Reboot is required to complete the installation.
```

更新完畢後，需要重新開機，指令如下：
`shutdown reboot -r "patch reboot"`

#### Passive node
當 Witness node 更新完畢後，再登入到 Passive node (`10.1.0.2`)上。
> 登入後，不用切到 _shell_ 裡，在 _appliancesh_ 即可。

將要更新的 patch 檔，掛載到 Passive node 上，並輸入以下指令。
`software-packages install --iso --acceptEulas`

更新完畢後，需要重新開機，指令如下：
`shutdown reboot -r "patch reboot"`

#### Last node
當 Passive node 及 Witness node 更新完畢並正常運作後，登入到 vSphere Web Client ，進行手動的 failover，把 Active node 切換成 Passive node。(需要等待一段時間)
> 此時的IP為：
> - vCenter MGT IP： `10.1.0.10`。
> - Passive node： `10.1.0.1`。
> - Active node： `10.1.0.2`。
> - Witness node： `10.1.0.3`。

切換完畢後(可登入vSphere Web Client確認)，以 ssh 的方式登入到 vCenter。
(此時進入shell以指令 `ip a` ，顯示的 IP 會是 `10.1.0.2`)

登入到 Passive node (`10.1.0.1`)上。
> 登入後，不用切到 _shell_ 裡，在 _appliancesh_ 即可。

將要更新的 patch 檔，掛載到 Passive node 上，並輸入以下指令。
`software-packages install --iso --acceptEulas`

更新完畢後，需要重新開機，指令如下：
`shutdown reboot -r "patch reboot"`

### 解除 Maintenance Mode
待 Last node 更新完畢後，登入到 vSphere Web Client，將 vCenter 的 Maintenance Mode 解除。

即更新完畢。

## 後記
更新後還是抓不到，已哭。

## Reference
- [vSphere 6.5 - Updating Appliance configured with vCenter HA](https://davidstamen.com/2017/02/03/vsphere-65-updating-appliance-configured-with-vcenter-ha/)