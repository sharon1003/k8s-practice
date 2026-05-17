## Goal
做一個簡單的python程式，然後包裝成 Docker Image

### 第一步：建一個資料夾
```
mkdir ~/k8s-practice
cd ~/k8s-practice
```

### 第二步：建一個簡單的 Python 檔案
``` bash
cat > app.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello from my first container!"}
EOF
```

### 第三步：建一個 Dockerfile

``` bash
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
RUN pip install fastapi uvicorn
COPY app.py .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 第四步，確認兩個檔案都在(app.py, Dockerfile)

### 說明 Dockerfile 程式碼
```FROM python:3.11-slim``` 
從 Python 3.11 的 Image 開始（別人已經做好的基底，不用從零開始）

```WORKDIR /app```
進到 Container 裡面的 /app 資料夾（之後的指令都在這裡執行）

```RUN pip install fastapi uvicorn```
安裝需要的套件

```COPY app.py .```
把你的 app.py 複製進 Container 裡面，注意app.py後面有 . 表示當前資料夾

```CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]```
Container 啟動時執行的第一個指令

```
FROM     → 用哪種鍋子開始煮（基底）
WORKDIR  → 在哪個廚房工作
RUN      → 備料（安裝套件）
COPY     → 把食材放進去（你的程式碼）
CMD      → 開始煮（啟動程式）
```

---

#### There are no dump questions
Container 裡面的 /app 資料夾（之後的指令都在這裡執行）是什麼意思？
Container 裡面是一個完整的 Linux 系統，有自己的資料夾結構
```
Container 內部
/
├── bin/
├── etc/
├── home/
├── app/          ← WORKDIR 指定的地方
│   └── app.py    ← COPY 進來的檔案會在這裡
└── usr/

WORKDIR /app      # 先進到 /app 資料夾
COPY app.py .     # 把 app.py 複製到「這裡」（也就是 /app/app.py）
```

---

### build 第一個 Image
``` docker build -t my-first-api:v1 .```
* ```-t my-first-api:v1``` : 幫 Image 取名子 (-t tag)
* ```.```: 在目前這個資料夾找 Dockerfile

### build 的過程
```
[1/4] FROM python:3.11-slim      ← 下載基底 Image
[2/4] WORKDIR /app               ← 建立 /app 資料夾
[3/4] RUN pip install fastapi    ← 安裝套件
[4/4] COPY app.py .              ← 把你的程式碼複製進去
```

### 確認 Image 產生
```docker images | grep my-first-api
my-first-api:v1    ac5656effacb        261MB         59.9MB
```

### 開始跑
```
docker run -d -p 8000:8000 --name my-first-api my-first-api:v1
```

跑完後，打開瀏覽器 http://localhost:8000

### 整理
```
1. 寫 app.py          ← 你的程式
2. 寫 Dockerfile      ← 食譜
3. docker build       ← 照食譜做出 Image
4. docker run         ← 把 Image 跑成 Container
5. 瀏覽器看到結果        ← Container 正常運作
```
