# O-RAN ONAP-Auto-Deploy 實作版繁中
- Automated deployment and testing Using SMO package and ONAP Python SDK
- 簡報：https://reurl.cc/oZLy9D

## SMO Package based on ONAP
SMO package 可在 “it/dep” repository 中的 ORAN gerrit 中訪問：
https://gerrit.o-ran-sc.org/r/gitweb?p=it/dep.git;a=tree;f=smo-install;h=2e4539d6c3c2e2a274d1913c89df371c956f0793;hb=HEAD 

它是基於 ONAP OOM repository，因為它被作為 Git Submodule。
ONAP chart 在使用時不會改變，也不會重新定義，而是透過使用 Helm 覆蓋機制進行設定。

ORAN chart 被主要用來定義 NON-RT RIC 的部分，其他圖表可以稍後增加。
定義的 Test chart 包含 network simulators（DU/RU/Topology server）、jenkins 或 python SDK 測試的 helm chart。

ChartMuseum 用於儲存本地構建的 chart（因為目前無法遠程獲取 ONAP chart 和 ORAN chart）

![image](https://user-images.githubusercontent.com/84045975/198906763-b04fb74e-300b-4b4b-a687-9dd542f521b2.png)

SMO package 包含一些 scripts 來設定 Node、安裝 smo/jenkins、啟動模擬器、解除安裝等….
這些腳本已分為 3 個不同的層，但根據您的設定，可以跳過一些腳本。
- 第 0 層：設定 Node（microk8s、helm、chartmuseum、測試工具等…）
- 第 1 層：構建 helm chart 並將其上傳到 ChartMuseum
- 第 2 層：部署具有特定 flavor 的 SMO、部署 network simulators、部署 CI/CD 工具等......

## Architecture
下圖從 high level 往下描述了環境以及使用此設定部署的各種實體(entities)

![image](https://user-images.githubusercontent.com/84045975/198906927-d7e07a84-4292-412a-ae1c-65f2856df330.png)

###### O-RAN SMO Deployment
# O-RAN SMO安裝

> SMO Based on ONAP

## 安裝先決條件
### 硬體需求
```
CPU: 6 Core
RAM:16GB
Storge:50GB
```
### 軟體環境需求
測試環境 OS：Ubuntu 22.04(伺服器版)
```
$ docker --version
Docker version 24.0.6

$ docker-compose version
docker-compose version v2.20.3
docker-py version: 5.0.3
CPython version: 3.10.12
OpenSSL version: OpenSSL 3.0.2 15 Mar 2022

$ git --version
git version 2.34.1
```
# 安裝前置工作
## 安裝 git、json輕量級的模擬器和py3
```cmd=
sudo apt-get update 
sudo apt-get install git
sudo apt install python3-pip
```
```
sudo apt-get install jq
```
## clone the repository
```
git clone https://gerrit.o-ran-sc.org/r/oam.git -b dawn
```
## 建置Docker環境
> [參照資料](https://docs.docker.com/engine/install/ubuntu/) 別擔心全部複製貼上就可以安裝了~

首先先更新 apt-get
```bash
sudo apt-get update
```
安裝需要的 packages curl 之類的 喵
```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
新增官方 Docker GPG 密鑰
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
新增 穩定版本的 repository
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
## 安裝Docker Engine
還是一樣先更新 Docker 相關套件
安裝最新版本的 engine,command and container
```bash
sudo apt-get update;sudo apt-get install docker-ce docker-ce-cli containerd.io
```

跑跑看 Docker hello world 看看有沒有出現(第一次會需要下載)
```bash
sudo docker run hello-world
```

然後你可以使用其他 Docker 指令來查看你的 Image 和 Container
> 推薦閱讀：[Docker 基本指令介紹 ](https://hackmd.io/@thc1006/rku088xM5)[color=#F4B400]

輸入以下指令測試安裝是否成功 (會出現你的 Docker 版本)
```bash
docker --version
```
![](https://i.imgur.com/b5C67tt.png)

### 提高權限
將使用者加入群組，來提高權限執行 Docker，
讓之後的安裝過程，系統才不會一直跟你靠腰你沒有權限！

首先新增一個 Docker 群組
```cmd=
sudo groupadd docker
```
然後將使用者加入群組

> 更改下方命令列 XXXX 的部分，改成你的使用者名稱
如：root@192.168.0.1 (root 就是你的使用者名稱)[color=#F4B400]
```cmd=
sudo usermod -aG docker XXXX
```
## 獨立安裝docker-compose
教程：https://docs.docker.com/compose/install/standalone/

* Troubleshooting
> 如果安裝遇到 Permission denied 的問題，請參閱下方連結(提高權限)：
https://stackoverflow.com/questions/59265190/permission-denied-in-docker-compose-on-linux
[color=#F4B400]

驗證安裝版本
```cmd=
docker compose version
```
# 開始安裝輕量版SMO
> 本教程以 O-RAN Release D 作為示範(穩定版本)[color=#F4B400]

### 參考資料：
* [D-Release Integration - Test Environment](https://wiki.o-ran-sc.org/display/OAM/D-Release+Integration+-+Test+Environment)
* [O1 Simulator and lightweight ONAP based SMO deployment](https://wiki.opennetworking.org/display/COM/O1+Simulator+and+lightweight+ONAP+based+SMO+deployment)

## 修改/etc/hosts
```
cat /etc/hosts
```
你會看到如下的資訊:
```
127.0.0.1               localhost
127.0.1.1               <your-system>
<deloyment-system-ipv4> sdnc-web <your-system>
<deloyment-system-ipv4> identity <your-system>
```
那我們要做一些修改變成像是下面那樣
> 用 nano 或是 vim 都來修改都可以[color=#F4B400]
```
nano /etc/hosts
```
```
127.0.0.1       localhost
127.0.1.1       oran-server
172.17.0.1   sdnc-web  oran-server
172.17.0.1   identity  oran-server
```
* "oran-server" 是我系統的名稱，你可以改成自己系統的名稱。
* "172.17.0.1" 是我的"deloyment-system-ipv4"，你要改成你自己的 host IP。

## 開始安裝solution
注意 會需要按照流程進行安裝
```cmd=
cd oam/solution/integration
sudo docker-compose -f smo/common/docker-compose.yml up -d
# 等待container安裝
python3 smo/common/identity/config.py
sudo docker-compose -f smo/onap-policy/docker-compose.yml up -d
sudo docker-compose -f smo/oam/docker-compose.yml up -d
sudo docker-compose -f smo/non-rt-ric/docker-compose.yml up -d
```
> Troubleshooting(你可能會收到 rapp pull 失敗的警告)[color=#ff0000]	

![](https://hackmd.io/_uploads/HJPlCnaph.png)

這時你就輸入下列這個指令，手動 pull rapp 就解決問題了~
```
sudo docker pull nexus3.o-ran-sc.org:10002/o-ran-sc/nonrtric-plt-rappcatalogue:1.1.0
```
可能會需要等待兩分鐘，讓所有服務正常啟動~

> 如果你想要為 O-RU closed loop recovery usecase 建立/部署 Apex 策略，這裡有相關 Test case 的補充資料：[color=#F4B400]
> - Create/Deploy the Apex policy for O-RU closed loop recovery Use Case
> - 參考資料：https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=35881325

# populate data into Non-RT-RIC
有關如何運行 Non-RT-RIC 的完整說明，可以在此頁面中找到：
https: //wiki.o-ran-sc.org/display/RICNR/Release+D

> 將 data 安裝入 Non-RT 的原因是，由於 Non-RT RIC 的安裝是 optional，所以這邊需要手動安裝其內部 component 以進行測試和 Demo。 [color=#F4B400]

安裝在 non-rt-ric/data directory 路徑底下
```
cd oam/solution/integration/smo/non-rt-ric/data
```
### Prepare Pmsdata
Script <code class="code">preparePmsData.sh</code> 會向policy-agent service 傳送 http 請求，並建立對應的 policy instances。
```
# run the script
bash preparePmsData.sh
```
Output be like:
```
root@OAM:~/test/oam/solution/integration/smo/non-rt-ric/data# bash preparePmsData.sh 
using policy_agent port: 8091
using a1-sim-OSC port: 30001
using a1-sim-STD port: 30005
using protocol: http


policy agent status:
hunky dory200


ric1 version:
OSC_2.1.0200


ric2 version:
STD_2.0.0200


create policy type 1 to ric1:
Policy type 1 is OK.201


create policy type 2 to ric2:
Policy type 2 is OK.201


policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":[]}200
policy types from policy agent:
{"policytype_ids":["1","2"]}200


create service ric-registration to policy agent:
201


create policy aa8feaa88d944d919ef0e83f2172a5000 to ric1 with type 1 and service controlpanel via policy agent:
201


policy numbers from ric1:
1200


create policy aa8feaa88d944d919ef0e83f2172a5100 to ric2 with type 2 and service controlpanel via policy agent:
201


policy numbers from ric2:
1200


policy id aa8feaa88d944d919ef0e83f2172a5000 from policy agent:
200


policy id aa8feaa88d944d919ef0e83f2172a5100 from policy agent:
200
```

### Prepare dmaapMsg
Script <code class="code">prepareDmaapMsg.sh</code> 會向 dmaap message router 傳送訊息，然後 Non-RT RIC policy-agent service 從 dmaap poll messages，並建立相應的 policy instances。
```
bash prepareDmaapMsg.sh
```
Output be like:
```
❯ bash prepareDmaapMsg.sh                                                                    
using dmaap-mr port: 3904
using a1-sim-OSC port: 30001
using a1-sim-STD port: 30003
using a1-sim-STD-v2 port: 30005
using protocol: http


dmaap-mr topics:
{"topics": [
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-WRITE"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-PDP-PAP"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-NOTIFICATION"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "__consumer_offsets"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-READ"
    }
]}200

dmaap-mr create topic A1-POLICY-AGENT-READ:
204

dmaap-mr create topic A1-POLICY-AGENT-WRITE:
204

dmaap-mr topics:
{"topics": [
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-WRITE"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-PDP-PAP"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-NOTIFICATION"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "__consumer_offsets"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-READ"
    }
]}200

ric1 version:
OSC_2.1.0200

ric2 version:
STD_1.1.3200

ric3 version:
STD_2.0.0200

create policy type 1 to ric1:
The policy type already exists and instances exists400

create policy type 2 to ric3:
The policy type already exists and instances exists400

policy types from policy agent:
["1","2"]200


create service 1 to policy agent via dmaap_mr:
{
    "serverTimeMs": 2,
    "count": 3
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   439  100   439    0     0    544      0 --:--:-- --:--:-- --:--:--   543
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998500\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998501\",\"originatorId\":\"849e6c6b421\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"201 CREATED\"}"
]


create policies to ric1 & ric2 & ric3 with type1 and service1 via dmaa_mr:
{
    "serverTimeMs": 1,
    "count": 5
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   215  100   215    0     0    884      0 --:--:-- --:--:-- --:--:--   884
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363459196978900\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:39.197067\",\"status\":\"200 OK\"}"
]


get policy from policy agent via dmaap_mr:
{
    "serverTimeMs": 0,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   218  100   218    0     0   3694      0 --:--:-- --:--:-- --:--:--  3694
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304565904621535\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:05.905035\",\"status\":\"201 CREATED\"}"
]


create service 2 to policy agent via dmaap_mr:
{
    "serverTimeMs": 1,
    "count": 3
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   218  100   218    0     0   5317      0 --:--:-- --:--:-- --:--:--  5317
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253556\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"201 CREATED\"}"
]


create policies to ric1 & ric2 & ric3 with type1 and service1 via dmaa_mr:
{
    "serverTimeMs": 0,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   218  100   218    0     0   5589      0 --:--:-- --:--:-- --:--:--  5589
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253557\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"201 CREATED\"}"
]


get policy from policy agent via dmaap_mr:
{
    "serverTimeMs": 3,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   220  100   220    0     0   4230      0 --:--:-- --:--:-- --:--:--  4230
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253558\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"404 NOT_FOUND\"}"
]


policy numbers from ric1:
4200

policy numbers from ric2:
0200

policy numbers from ric3:
1200
```

等待幾分鐘後再執行一次:
```
bash prepareDmaapMsg.sh 
```
Output be like:
```                                                                     
using dmaap-mr port: 3904
using a1-sim-OSC port: 30001
using a1-sim-STD port: 30003
using a1-sim-STD-v2 port: 30005
using protocol: http


dmaap-mr topics:
{"topics": [
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-WRITE"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-PDP-PAP"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-NOTIFICATION"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "__consumer_offsets"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-READ"
    }
]}200

dmaap-mr create topic A1-POLICY-AGENT-READ:
204

dmaap-mr create topic A1-POLICY-AGENT-WRITE:
204

dmaap-mr topics:
{"topics": [
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-WRITE"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-PDP-PAP"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "POLICY-NOTIFICATION"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "__consumer_offsets"
    },
    {
        "owner": "",
        "txenabled": false,
        "topicName": "A1-POLICY-AGENT-READ"
    }
]}200

ric1 version:
OSC_2.1.0200

ric2 version:
STD_1.1.3200

ric3 version:
STD_2.0.0200

create policy type 1 to ric1:
The policy type already exists and instances exists400

create policy type 2 to ric3:
The policy type already exists and instances exists400

policy types from policy agent:
["1","2"]200


create service 1 to policy agent via dmaap_mr:
{
    "serverTimeMs": 0,
    "count": 3
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   220  100   220    0     0  11000      0 --:--:-- --:--:-- --:--:-- 11578
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253558\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"404 NOT_FOUND\"}"
]


create policies to ric1 & ric2 & ric3 with type1 and service1 via dmaa_mr:
{
    "serverTimeMs": 0,
    "count": 5
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   677  100   677    0     0  29434      0 --:--:-- --:--:-- --:--:-- 30772
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3000},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3000\\\",\\\"ueId\\\":\\\"ue3000\\\"}}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3100},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3100\\\",\\\"ueId\\\":\\\"ue3100\\\"}}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}"
]


get policy from policy agent via dmaap_mr:
{
    "serverTimeMs": 0,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4470  100  4470    0     0   128k      0 --:--:-- --:--:-- --:--:--  128k
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3200},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3100\\\",\\\"ueId\\\":\\\"ue3100\\\"}}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"type\\\":\\\"about:blank\\\",\\\"status\\\":404,\\\"detail\\\":\\\"org.onap.ccsdk.oran.a1policymanagementservice.exceptions.EntityNotFoundException: Could not find policy: 0f7bb041e1584b1fa17e87520d70a3102\\\"}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"404 NOT_FOUND\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998500\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998501\",\"originatorId\":\"849e6c6b421\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363459196978900\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-02 10:17:39.197067\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304565904621535\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:05.905035\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253556\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253557\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304566656253758\",\"originatorId\":\"849e6c6b422\",\"type\":\"response\",\"message\":\"\",\"timestamp\":\"2020-08-01 17:56:06.656949\",\"status\":\"201 CREATED\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"policy_id\\\":\\\"0f7bb041e1584b1fa17e87520d70a3000\\\",\\\"policytype_id\\\":\\\"1\\\",\\\"ric_id\\\":\\\"ric1\\\",\\\"policy_data\\\":{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3000.0},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3000\\\",\\\"ueId\\\":\\\"ue3000\\\"}},\\\"service_id\\\":\\\"service1\\\",\\\"transient\\\":false,\\\"status_notification_uri\\\":\\\"\\\"}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"policy_id\\\":\\\"0f7bb041e1584b1fa17e87520d70a3100\\\",\\\"policytype_id\\\":\\\"1\\\",\\\"ric_id\\\":\\\"ric1\\\",\\\"policy_data\\\":{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3100.0},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3100\\\",\\\"ueId\\\":\\\"ue3100\\\"}},\\\"service_id\\\":\\\"service1\\\",\\\"transient\\\":false,\\\"status_notification_uri\\\":\\\"\\\"}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"policy_id\\\":\\\"0f7bb041e1584b1fa17e87520d70a3101\\\",\\\"policytype_id\\\":\\\"1\\\",\\\"ric_id\\\":\\\"ric1\\\",\\\"policy_data\\\":{\\\"qosObjectives\\\":{\\\"priorityLevel\\\":3200.0},\\\"scope\\\":{\\\"qosId\\\":\\\"qos3100\\\",\\\"ueId\\\":\\\"ue3100\\\"}},\\\"service_id\\\":\\\"service1\\\",\\\"transient\\\":false,\\\"status_notification_uri\\\":\\\"\\\"}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"200 OK\"}",
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596304567017739720\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"{\\\"type\\\":\\\"about:blank\\\",\\\"status\\\":404,\\\"detail\\\":\\\"org.onap.ccsdk.oran.a1policymanagementservice.exceptions.EntityNotFoundException: Could not find policy: 0f7bb041e1584b1fa17e87520d70a3102\\\"}\",\"timestamp\":\"2020-08-01 17:56:07.017744\",\"status\":\"404 NOT_FOUND\"}"
]


create service 2 to policy agent via dmaap_mr:
{
    "serverTimeMs": 17,
    "count": 3
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   215  100   215    0     0   6142      0 --:--:-- --:--:-- --:--:--  6142
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998500\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"200 OK\"}"
]


create policies to ric1 & ric2 & ric3 with type1 and service1 via dmaa_mr:
{
    "serverTimeMs": 6,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   215  100   215    0     0   3839      0 --:--:-- --:--:-- --:--:--  3839
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363458549998501\",\"originatorId\":\"849e6c6b421\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:38.550324\",\"status\":\"200 OK\"}"
]


get policy from policy agent via dmaap_mr:
{
    "serverTimeMs": 0,
    "count": 4
}200

get result from mr of previous request:
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   215  100   215    0     0   9772      0 --:--:-- --:--:-- --:--:--  9772
[
  "{\"requestId\":\"23343221\",\"correlationId\":\"1596363459196978900\",\"originatorId\":\"849e6c6b420\",\"type\":\"response\",\"message\":\"OK\",\"timestamp\":\"2020-08-02 10:17:39.197067\",\"status\":\"200 OK\"}"
]


policy numbers from ric1:
7200

policy numbers from ric2:
0200

policy numbers from ric3:
2200
```

### Prepare Ecs Data
Script <code class="code">prepareIcsData.sh</code> 向 ics service 發送 http 請求，並建立相應的資料。

```
# 執行這個 bash
bash prepareEcsData.sh
```
Output be like:
```
root@OAM:~/oam/solution/integration/smo/non-rt-ric/data# bash prepareEcsData.sh
using ecs port: 8083
using protocol: http


ECS status:
{"status":"hunky dory","no_of_producers":0,"no_of_types":0,"no_of_jobs":0} 200

Create EiType:
201

Get EiTypes:
[
  "type1"
]
200


Get Individual EiType:
{
  "info_job_data_schema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "title": "STD_Type1_1.0.0",
    "description": "EI-Type 1",
    "type": "object"
  }
}
200


Create EiProducer:
201

Get EiProducers:
[
  "1"
]
200


Get Individual EiProducer:
{
  "supported_info_types": [
    "type1"
  ],
  "info_job_callback_url": "http://producer:80/callbacks/job/prod-a",
  "info_producer_supervision_callback_url": "http://producer:80/callbacks/supervision/prod-a"
}
200


Get Individual EiProducer:
{
  "operational_state": "ENABLED"
}
200


Create EiJob Of A Certain Type type1:
201

Get EiJobs:
[
  "job1"
]
200


Get Individual EiJob:
{
  "eiTypeId": "type1",
  "jobOwner": "ricsim_g3_1",
  "jobDefinition": {
    "jobparam1": "value1_job1",
    "jobparam2": "value2_job1",
    "jobparam3": "value3_job1"
  },
  "jobResultUri": "https://ricsim_g3_1:8185/datadelivery",
  "jobStatusNotificationUri": "http://producer:80/"
}
200
```

開啟瀏覽器: http://localhost:8182/ 
應該就可以看到以下的畫面

> 我是用 VS code SSH 連線進行開發，所以如果IDE沒有跳出轉發通知，就會需要我們手動轉發連接埠(port:8182) [color=#F4B400]

![](https://hackmd.io/_uploads/S1FkHlC-T.png =70%x)

![](https://hackmd.io/_uploads/r1blrgRW6.png =70%x)

![](https://hackmd.io/_uploads/SJNxrl0Z6.png =70%x)

> 可以更改 yaml 檔來調整 Non-RT RIC 的 Policy


## Simulated O-DU and O-RU according to O-RAN Hybrid Architecture
```cmd=
cd oam/solution/integration
docker-compose -f network/docker-compose.yml up -d
```

```cmd=
docker-compose -f network/docker-compose.yml restart ntsim-ng-o-du-1122
#必須重新啟動2次 container 才可以正式被加入管理列表
docker-compose -f network/docker-compose.yml restart ntsim-ng-o-du-1122
```
```
python3 network/config.py 
```
```
$ python3 network/config.py                          
# 會跑出下列資訊
/usr/lib/python3/dist-packages/urllib3/connectionpool.py:999: InsecureRequestWarning: Unverified HTTPS request is being made to host 'localhost'. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  warnings.warn(
Set highstreet-O-RU-11222 True
/usr/lib/python3/dist-packages/urllib3/connectionpool.py:999: InsecureRequestWarning: Unverified HTTPS request is being made to host 'localhost'. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  warnings.warn(
Set highstreet-O-RU-11221 True
/usr/lib/python3/dist-packages/urllib3/connectionpool.py:999: InsecureRequestWarning: Unverified HTTPS request is being made to host 'localhost'. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  warnings.warn(
Set highstreet-O-DU-1122 True
/usr/lib/python3/dist-packages/urllib3/connectionpool.py:999: InsecureRequestWarning: Unverified HTTPS request is being made to host 'localhost'. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#ssl-warnings
  warnings.warn(
Set highstreet-O-RU-11223 True
```

> 如都顯示 “True” 表示通過 SDNR(SMO) 對 NETCONF server(NF)的 config成功了。

## 開啟O1 Dashboard Web UI(ONAP based)

瀏覽器輸入：https://<your-public-ip>:8453/
```cmd=
username: admin
password: Kp8bJ4SXszM0WXlhak3eHlcse2gAw84vaoGGmJvUy2U 
```
就可以看到 SMO 的畫面了~ 
    
![](https://hackmd.io/_uploads/rk5tnw116.png)

## solution終止（請依順序輸入指令）
```
sudo docker-compose -f network/docker-compose.yml down
sudo docker-compose -f smo/oam/docker-compose.yml down
sudo docker-compose -f smo/onap-policy/docker-compose.yml down
sudo docker-compose -f smo/non-rt-ric/docker-compose.yml down
sudo docker-compose -f smo/common/docker-compose.yml down
```
## 參考資料
* [D-Release Integration - Test Environment](https://wiki.o-ran-sc.org/display/OAM/D-Release+Integration+-+Test+Environment)
* [O1 Simulator and lightweight ONAP based SMO deployment](https://wiki.opennetworking.org/display/COM/O1+Simulator+and+lightweight+ONAP+based+SMO+deployment)
* [O-RAN SC Release D - Build](https://wiki.o-ran-sc.org/display/RICNR/Release+D+-+Build)
* [Closed Loop Use Case Testing](https://wiki.o-ran-sc.org/pages/viewpage.action?pageId=35881638)

### 4. Demo Recording	
Plan:
- What is the SMO package 
  - smo-package1：https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package1.mp4?version=1&modificationDate=1648066446155&api=v2
  - smo-package2: https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package2.mp4?version=1&modificationDate=1648066105178&api=v2
  - smo-package3: https://wiki.o-ran-sc.org/download/attachments/47746045/.xdp_smo-package3.mp4?version=1&modificationDate=1648066291100&api=v2
- What is ONAP Python SDK
ONAP-Python-SDK4: https://wiki.o-ran-sc.org/download/attachments/47746045/onappython-sdk.mp4?version=1&modificationDate=1647618959002&api=v2
  - What is ORAN Python SDK
  - How to execute the SMO usecase tests
  - SMO Nomad/ephemeral CICD jenkins

## 參考
https://wiki.o-ran-sc.org/display/OAM/OAM+Architecture#
