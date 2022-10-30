# ONAP-Auto-Deploy
- Automated deployment and testing Using SMO package and ONAP Python SDK
- 簡報：https://reurl.cc/oZLy9D

## Driver
自動化是當今的關鍵，部署和測試 需要具備可重複性及可移植性。
為了使用 ORAN SC 軟體實現一定程度的自動化，我們採取了以下步驟：
- 創建一個簡單的部署方法，重用 ONAP 已完成的工作：參見 SMO 包 (https://jira.onap.org/browse/REQ-887)
- 重用在 ONAP 中已經成功使用的測試自動化工具：參見 Python-SDK (https://python-onapsdk.readthedocs.io/en/master/)
- 擴展部署機制，以提供一個獨立的、可移植的設定，以驗證各類型部署的使用範例（稱為“flavors”）
最終目標是為 O-RAN SC 社群提供一種以最低基本需求，來部署 SMO 及其測試環境的方法，最終此設定可用於 lab 以自動驗證程式碼更改，並直接在程式碼審查工具中報告問題。
本 wiki 中所描述的設定絕不是封閉的，由於所選的所有工具有靈活性，所以它可以輕鬆擴展。

## SMO Package based on ONAP
SMO 套件可在 “it/dep” 儲存庫中的 ORAN gerrit 上訪問：
https://gerrit.o-ran-sc.org/r/gitweb?p=it/dep.git;a=tree;f=smo-install;h=2e4539d6c3c2e2a274d1913c89df371c956f0793;hb=HEAD 

它是基於 ONAP OOM 儲存庫，因為它被用作 Git Submodule。ONAP 圖表沒有改變使用，也沒有重新定義，但顯然是通過使用 Helm 覆蓋機制進行設定的。

ORAN 圖表被主要用於定義部分 NON RT RIC ，其他圖表可以稍後增加。
定義的測試圖表包含網路模擬器（DU/RU/拓撲服務器）、jenkins 或 python SDK 測試的 helm 圖表。

ChartMuseum 用於儲存本地構建的圖表（因為目前無法遠程獲取 ONAP 和 ORAN 圖表）

![image](https://user-images.githubusercontent.com/84045975/198906763-b04fb74e-300b-4b4b-a687-9dd542f521b2.png)

SMO 套件包含一些腳本來設定Node、安裝 smo/jenkins、啟動模擬器、卸載等….
這些腳本已分為 3 個不同的層，但根據您的設定，可以跳過一些腳本。
- 第 0 層：設定Node（microk8s、helm、chartmuseum、測試工具等…）
- 第 1 層：構建 helm 圖表並將其上傳到 ChartMuseum
- 第 2 層：部署具有特定 flavor 的 SMO、部署網路模擬器、部署 CI/CD 工具等......

## Architecture
下圖從 high level 往下描述了環境以及使用此設定部署的各種實質個體

![image](https://user-images.githubusercontent.com/84045975/198906927-d7e07a84-4292-412a-ae1c-65f2856df330.png)

基礎環境需求，一個標準 VM 或 Kubernetes cluster 運行Ubuntu 的 Node (建議新版本)
（如有需要，cluster 也可以是遠程的，只要它可以從主機 VM 訪問）
儲存庫的自述文件中說明了如何部署佈局，簡而言之它由以下部分組成：
- helm 圖表用來部署
  - SMO,模擬器
  - 一個專用的嵌入式 jenkins 實例
  - Helm Chart 儲存庫，用於儲存用於設置的本地有用圖表
- 一套實用的腳本，協助 Ubuntu VM 或 Node 設置環境
- 一組執行 Python SDK 的測試用例
Jenkins 實例將充當 Meta 執行器來編排部署並執行一組測試。
此佈局為嵌入式 Jenkins 實例，提供了與集群交互並連接到遠程儲存庫所需的所有配置

## Deployment guide

硬體最低要求
- 配置和選項，主要的環境部屬，靈活度非常高。

最低系統配置：
- 1 個虛擬機或主機：
- 6 個或更多 CPU 核心
- 20G 內存
- 60G磁盤（主要用於存儲容器鏡像
- Ubuntu 20.04 LTS（使用 18.04 LTS 進行測試）
### 須執行命令

```bash
$ cat /etc/os-release
NAME="Ubuntu" 
VERSION="18.04.6 LTS (Bionic Beaver)“
ID=ubuntu ID_LIKE=debian 
PRETTY_NAME="Ubuntu 18.04.6 LTS" 
VERSION_ID="18.04" 
HOME_URL="https://www.ubuntu.com/" SUPPORT_URL="https://help.ubuntu.com/" BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/" PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy" 
VERSION_CODENAME=bionic 
UBUNTU_CODENAME=bionic
```
小提醒：
確保您的使用者帳戶有足夠的權限來執行下列的命令，有些可能會需要您授予 sudo 權限換到有權限的帳戶，以執行這些命令。

### 執行命令
```bash
$ git --version
git version 2.17.1
```
### 安裝
從以下 gerrit 下載 it/dep 儲存庫：
執行下列三條命令
- $ mkdir workspace && cd workspace
- $ git clone --recurse-submodules https://gerrit.o-ran-sc.org/r/it/dep.git
- $ cd dep

請注意，您需要添加 recurse-submodules flag 因為部分的 git submodules 要指向已存在的相關圖表（ONAP）。
安裝非常簡單，儲存庫中提供了幾個實用程式腳本，允許用戶獨立設定所有/部分組件。
請查看位於 dep/smo-install 下的嵌入式自述文件，它提供了內容的完整描述以及如何設定各種flavors的環境。
https://gerrit.o-ran-sc.org/r/gitweb?p=it/dep.git;a=blob_plain;f=smo-install/README.md;hb=refs/heads/master

## User guide
本節介紹如何訪問和使用已部署的設定，
它允許用戶配置、運行和分析 ORAN SC 套件上的測試結果，也可以作為傳入程式碼更改的驗證工具。
### 1.訪問
系統一啟動，一個名為“test”的 namespace 應該出現在您的 Kubernetes 環境中。應該就能夠打開瀏覽器訪問 jenkins 實例：
http://<你的 k8s 主機>:32080/
![image](https://user-images.githubusercontent.com/84045975/198907241-981abe1e-2c74-4a65-bec4-2a630b556ca9.png)
所有作業都由部署自動配置，以便能夠：

連接到遠端儲存庫（提供來自 Github 和 Gerrit 的示例）
自動發現分支並打開 pull  請求/ Gerrit reviews
從儲存庫下載更改，構建部署和測試
也可以手動觸發啟動/停止/測試 SMO 套件（名稱中帶有 Manual 關鍵字的 Job）

可以使用預設 帳號/密碼（測試/測試）登錄到 Jenkins 實例（查看/修改配置） 
- 預設 帳號/密碼 可以在覆寫檔案中被覆寫（參見上面的安裝部分）	
### 2.運行測試
要執行測試，有 2 個選項：
自動排定檢查
設定對 github/gerrit 儲存庫的訪問，並讓系統掃描可以用的 branch，可以透過按一下 “-verify” 來查看已掃描到的內容。

一旦設定了對儲存庫的訪問權限，將自動創建新作業來驗證可使用的branch，觸發條件：你讓 job 啟動，他們就會自動來驗證。
或者
Job 被自動創建在 Gerrit reviews（gerrit）/ pull Requests（github），就會將自動檢查新更動，或者可以手動啟動。	

看下面的實際運行情況
![image](https://user-images.githubusercontent.com/84045975/198907285-941011ca-8bb9-4f0d-b4d5-ad6d999fce6f.png)
要手動操作時，會需要登錄（剛剛那個覆寫檔案的預設帳密

手動操作時，你可以提供儲存庫中 可以被 pull 的分支，以及 定義運行組件和測試覆寫文件 (Flavor) 的位置
執行測試後，可以通過 Jenkins UI 來追蹤 pipeline execution 的狀況可以透過作業執行 UI 中可用的 workspace 訪問 Logs/results ，也可以通過 Jenkins 介面來訪問執行歷史記錄、趨勢和步驟詳細資訊
![image](https://user-images.githubusercontent.com/84045975/198907309-ad1f0630-e5f7-4993-bd9d-49a3dbb8e389.png)

### 3.了解測試
工具運行兩組測試：
Unit Test 它們通常在 Unit 的級別來用來驗證 ORAN SDK 方法，每次 ORAN SDK 添加新方法時，都必須通過簡單的單元測試來進行驗證。
透過 pyTest 框架來執行。
整合測試
這些測試使用 ONAP/ORAN Python SDK 來驗證特定的 SMO 用例。 
他們可以初始化、提供、驗證、觸發等......，這些由　SMO 套件啟動的不同 ONAP/ORAN 組件。 
它們需要在 Kubernetes clusters 內或直接在 Kubernetes Node 上執行，因為測試需要訪問由 Kubernetes clusters 創建的虛擬網絡。

### 4. Demo Recording	
Plan:
What is the SMO package 
smo-package1：https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package1.mp4?version=1&modificationDate=1648066446155&api=v2
smo-package2: https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package2.mp4?version=1&modificationDate=1648066105178&api=v2
smo-package3: https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package3.mp4?version=1&modificationDate=1648066291100&api=v2
What is ONAP Python SDK
ONAP-Python-SDK4: https://wiki.o-ran-sc.org/download/attachments/47746045/onappython-sdk.mp4?version=1&modificationDate=1647618959002&api=v2
- What is ORAN Python SDK
- How to execute the SMO usecase tests
- SMO Nomad/ephemeral CICD jenkins

## 參考
https://wiki.o-ran-sc.org/display/OAM/OAM+Architecture#






