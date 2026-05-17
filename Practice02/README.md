# Day 2

如果你的 API 同時有 1000 個人在用，一個 Container 可能會撐不住。你覺得要怎麼解決？
> My Ans: 分給其他人

```
1000 個請求進來
        ↓
k8s Service（交通指揮）
    ↙    ↓    ↘
Pod1  Pod2  Pod3   ← 三個一樣的 Container 同時跑
```

每個 Pod 各自處理一部分請求，不會有單一 Container 被壓垮的問題。

*但這裡有一個問題*
三個 Pod 都跑同一個 my-first-api:v1 Image，外面的人怎麼知道要連哪一個？
總不能叫使用者自己決定「我要連 Pod 1」吧——而且 Pod 的 IP 會變，今天是 Pod 1，明天可能換 Pod 3。
這就是 k8s Service 存在的原因——它是一個固定的入口，背後自動分流。

```
使用者只知道這一個地址
        ↓
   k8s Service
   （固定入口）
    ↙   ↓   ↘
  Pod  Pod  Pod
（背後怎麼分你不用管）
```

---

### 1. 使用k8s之前，需要裝兩個東西：
* minikube — 在你 Mac 上跑一個迷你的 k8s cluster（學習用）
* kubectl — 跟 k8s 說話的指令工具
> bashbrew install minikube kubectl

確認是否安裝成功
> minikube version
> kubectl version --client


### 2. 啟動 minikube
> minikube start --driver=docker

第一次跑會比較久（要下載東西），大概 2-3 分鐘。

```
CPUs=2, Memory=6100MB    ← minikube 用了你 Mac 的 2 核心和 6GB 記憶體
Kubernetes v1.35.1       ← 跑的 k8s 版本
Done!                    ← 成功
```

### 3. 確認 cluster 狀態

> kubectl get nodes

```
minikube 21:40:44
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   4m5s   v1.35.1
```

**說明**
```
NAME       → minikube（這個 Node 的名字）
STATUS     → Ready（健康，可以接受工作）
ROLES      → control-plane（這個 Node 是 k8s 的大腦）
AGE        → 4m5s（跑了 4 分鐘）
VERSION    → v1.35.1（k8s 版本）
```

#### control-plane 是什麼？
k8s cluster 有兩種角色
```
1. control-plane 大腦
    決定 Pod 要跑在哪裡
    監控所有 Pod 狀態
    接受你的 kubectl 指令

2. worker node 
    實際跑 Pod 的地方

```

目前環境
```
你的 Mac
└── Docker
    └── minikube container（k8s cluster）
        └── 還沒有任何 Pod 在跑
```


下一步，我們要把你剛才做的 my-first-api 部署到這個 cluster 裡。
但在這之前，有一個問題想問你：

你覺得 k8s 要怎麼知道要跑哪個 Image、要跑幾個、用哪個 port？
> My Ans: 設置條件，在什麼情況下給誰，可以依照目前container的附載狀態

* 設條件，要跑哪個 Image、幾個、用哪個 port」
→ 這個叫 YAML 設定檔
* 「依照負載狀態自動調整」
→ 這個叫 HPA（Horizontal Pod Autoscaler）

### 1. YAML 設定檔
k8s 不是用指令告訴他「去跑這個」，而是寫一個檔案說明「我的狀態是這樣」，k8s 會自己想辦法得到
```
我要 3 個 my-first-api 的 Pod 一直跑著
用 port 8000
如果掛掉就重啟
```
k8s 讀到這個之後，會自己去跑 3 個 Pod，其中一個掛掉會自動補一個，永遠維持 3 個。
你不用說「去跑」，你只說「我要的結果是這樣」——這個概念叫做宣告式（Declarative）。

```
Python（命令式）      k8s YAML（宣告式）
────────────────     ──────────────────
步驟 1：做這個        我要的結果是這樣
步驟 2：做那個        k8s 自己決定怎麼做
步驟 3：做這個
```


### 建立一 YAML file
這個 YAML 叫做 Deployment——告訴 k8s「我要幾個 Pod、跑什麼 Image」。

```yaml
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: first-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: first-api
  template:
    metadata:
      labels:
        app: first-api
    spec:
      containers:
      - name: first-api
        image: my-first-api:v1
        ports:
        - containerPort: 8000
EOF
```

思考一下：
有幾個關鍵字你先看看，說說你覺得每個是什麼意思：
> replicas: 3
> image: my-first-api:v1
> containerPort: 8000

> My ans: replicas: 3 複製三份、我的image file: name my-first-api:v1、container 的 port 開 8000也就是這三個node的入口嗎

```
外面的請求
    ↓
Service（真正的入口，之後建）
    ↓
Pod 1: containerPort 8000
Pod 2: containerPort 8000
Pod 3: containerPort 8000
```

labels and selector
```
selector:
  matchLabels:
    app: first-api    ← 我要管理標籤是 first-api 的 Pod

template:
  metadata:
    labels:
      app: first-api  ← 這些 Pod 的標籤是 first-api
```

### 啟動 yaml
> kubectl apply -f deployment.yaml

### 查看狀態
> kubectl get pods

```
minikube 21:52:47
NAME          READY   STATUS     RESTARTS   AGE
first-api-76b4c86d6d-9vwfb   0/1   ImagePullBackOff 0          19s
first-api-76b4c86d6d-jq74l   0/1   ImagePullBackOff   0          19s
first-api-76b4c86d6d-nj4t7  0/1   ImagePullBackOff   0          19s
```

#### ImagePullBackOff 是什麼？
k8s 去找 my-first-api:v1 這個 Image，但找不到——因為它預設去網路上找，但這個 Image 只在你的 Mac 本機。
```
k8s 去找 my-first-api:v1
        ↓
去 Docker Hub 找（網路）
        ↓
找不到！
        ↓
ImagePullBackOff
```

### 解法：Image 載入到 minikube
```
minikube image load my-first-api:v1
kubectl get pods
```

如果出現 ErrImagePull 代表還在跑


成功的話會出現
```
NAME                   READY   STATUS    RESTARTS   AGE
first-api-76b4c86d6d-9vwfb    1/1     Running   0          84s
first-api-76b4c86d6d-jq74l    1/1     Running   0          84s
first-api-76b4c86d6d-nj4t7    1/1     Running   0          84s
```

### 來測試 k8s 的自我修復能力
故意刪掉一個 Pod：
> kubectl delete pod lte-api-76b4c86d6d-9vwfb
然後馬上看：
> kubectl get pods
看看會發生什麼事，貼給我。