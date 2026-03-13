#  任務

## 任務說明

1. 以下為一份 TypeScript 的 Dockerfile，請說明有哪些方向可以優化此 Dockerfile。
```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY tsconfig.json ./
COPY src ./src
RUN npm run build
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

2. 嘗試使用 buildx 將 dockerfile（不需要是這一份，可以請 AI 根據你的習慣語言生一個範例） 編譯成多架構的 image，Image 需要可以分別在 x86 跟 ARM 上執行。
完成後請嘗試驗證是否有成功執行（可以開雲端 VM 執行看看）。

## 回覆說明

### 1. dockerfile 優化

```dockerfile
# 階段一：建置階段 (Builder)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
# 使用 npm ci 確保安裝乾淨且一致的依賴版本
RUN npm ci
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# 階段二：正式運行階段 (Production)
FROM node:20-alpine AS runner
WORKDIR /app

# 設定環境變數為 production，讓 Node.js 和套件以最佳化模式運行
ENV NODE_ENV=production

COPY package*.json ./
# 只安裝 production 需要的依賴
RUN npm ci --only=production

# 從建置階段複製編譯後的檔案
COPY --from=builder /app/dist ./dist

# 切換為非 root 使用者以提升安全性 (node 映像檔內建 node 使用者)
USER node

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

- 說明
  - Dockerfile 中使用 npm ci (Clean Install) 時，它會強迫 Node.js 嚴格依照 package-lock.json 裡紀錄的精確版本來安裝套件，以確保每次建置出來的環境 100% 一致。
  - 因為是手動建立了一個測試版本的 package.json，並沒有產生 package-lock.json，所以 npm ci 找不到這個檔案就會 ERROR。
  - 實際運作時是改用以下方式
    ```
    修改 Dockerfile，把裡面的兩處 npm ci ：
    第 6 行：將 RUN npm ci 改為 RUN npm install
    第 17 行：將 RUN npm ci --only=production 改為 RUN npm install --only=product
    ion 
    ```
### 2. 運作

以下使用 Podman 進行
- Podman 多架構下，要先建立 manifest
   ```powershell
   podman manifest create myapp
   ```
  - 執行 build ：
   ```powershell
   podman build --platform linux/amd64 --manifest myapp .
   podman build --platform linux/arm64 --manifest myapp .
   ```
 - 執行完成後，先確認 manifest 是否建立成功
   ```
   podman manifest inspect myapp
   ```

   成功會有類似以下的字串
     ```JSON
     {
      "manifests": [
        {
          "platform": {
            "architecture": "amd64",
            "os": "linux"
          }
        },
        {
          "platform": {
            "architecture": "arm64",
            "os": "linux"
          }
        }
      ]
    }
   ```
   
 - 查看本地 image
   通常會有，localhost/myapp，這是一個 manifest list
   ```powershell
   podman images
   ```
 - 最後執行 container
   Podman 會自動選適合 CPU 的架構
   ```powershell
   podman run -p 8080:80 localhost/myapp
   ```

### 其他紀錄
最初的練習未使用 --manifest，而是分開執行：
- amd64
 ```powershell
 podman build --platform linux/amd64 -t my-node-app:amd64 .
 ```
 <img width="1117" height="266" alt="image" src="https://github.com/user-attachments/assets/8c9b6c08-b0e3-4316-b0b1-712127bfb3dc" />

- arm64

 ```powershell
 # 要先讓 x86 電腦可以執行 ARM 容器，安裝 QEMU emulator
 podman run --privileged --rm docker.io/tonistiigi/binfmt --install all
 ```
 ```powershell
 podman build --platform linux/arm64 -t my-node-app:arm64 .
 ```

 <img width="1184" height="354" alt="image" src="https://github.com/user-attachments/assets/a28d0295-2184-4976-abb0-eed0302de539" />
