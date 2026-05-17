# Day 3 -- Service

先想一個問題再來寫 YAML。

你現在有三個 Pod 在跑：
Pod 1：IP 10.0.0.1
Pod 2：IP 10.0.0.2
Pod 3：IP 10.0.0.3
如果你想從瀏覽器連進去，你要連哪個 IP？

Ans: 
```
Pod IP 一直在變
        ↓
你不知道要連哪個 IP
        ↓
就算知道，Pod 重啟之後 IP 變了
        ↓
又要重新找
```

Service 是解法，給一個固定入口，不管 Pod IP 怎麼變，都可以分流到其中一個 pod

```
你連 Service（固定 IP）
        ↓
Service 自動找到有 lte-api 名牌的 Pod
        ↓
分流到其中一個 Pod
```

kind 的變化
```
Deployment  → 有 selector + label（既是老闆也是員工）
Service     → 只有 selector（只是老闆，負責找人分流）
Pod         → 只有 label（只是員工，等著被找）
```

```
port: 80         ← Service 對外開放的 port（外面連這個）
targetPort: 8000 ← Pod 裡面的 port（轉進去這個）
```
就像總機：
```
你打 80（總機號碼）
        ↓
Service 轉接到 Pod 的 8000
```

### apply
> kubectl apply -f service.yaml

結果
```
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
first-api-service   NodePort    10.111.125.108   <none>        80:30751/TCP   18s
kubernetes          ClusterIP   10.96.0.1   <none>        443/TCP        4d


NAME               → first-api-service（你的 Service）
TYPE               → NodePort（可以從外面連進來）
CLUSTER-IP         → 10.111.125.108（Service 的固定 IP，在 cluster 內部用）
EXTERNAL-IP        → none（minikube 環境沒有，真實雲端環境會有）
PORT(S)            → 80:30751（80 是 Service port，30751 是外部連進來的 port）
```

### 重新啟動
還是 ImagePullBackOff——舊的 Pod 還在用舊的 image 名字 my-lte-api:v1。
需要重新啟動 Pod 才會套用新的 deployment.yaml。
> kubectl rollout restart deployment first-api


### 開啟瀏覽器
> minikube service first-api-service


## 整體流程
```
寫 app.py + Dockerfile
        ↓
docker build → Image
        ↓
minikube image load
        ↓
deployment.yaml → 3 個 Pod
        ↓
service.yaml → 固定入口 + 分流
        ↓
瀏覽器連進來看到結果
```