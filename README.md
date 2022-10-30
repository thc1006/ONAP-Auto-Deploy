# ONAP-Auto-Deploy
Automated deployment and testing Using SMO package and ONAP Python SDK
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
![image](https://user-images.githubusercontent.com/84045975/198906763-b04fb74e-300b-4b4b-a687-9dd542f521b2.png =75%x)
SMO 套件包含一些腳本來設定Node、安裝 smo/jenkins、啟動模擬器、卸載等….
這些腳本已分為 3 個不同的層，但根據您的設定，可以跳過一些腳本。
- 第 0 層：設定Node（microk8s、helm、chartmuseum、測試工具等…）
- 第 1 層：構建 helm 圖表並將其上傳到 ChartMuseum
- 第 2 層：部署具有特定 flavor 的 SMO、部署網路模擬器、部署 CI/CD 工具等......





