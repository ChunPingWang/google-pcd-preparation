---
title: Lab 01 Cloud Run 端到端
tags:
  - gcp/pcd
  - lab
  - service/cloud-run
status: 未做
confidence: 1
預估時間: 60 分鐘
updated: 2026-09-27
---

# Lab 01：Cloud Run 端到端（部署 → canary → 回滾）

> [!abstract] 你會學到
> Cloud Run 的三層資源模型、從原始碼部署、服務帳戶與驗證、**流量分流與秒級回滾**、min/max instances 的效果。
> 對應筆記：[[Cloud Run]]、[[Section 3 設定雲端原生應用的部署]]

**前置**：已有 project、已裝 `gcloud`、已設 [[03 官方資源清單]] 提到的 **Budget Alert**。

---

## 0️⃣ 環境準備

```bash
export PROJECT=$(gcloud config get-value project)
export REGION=asia-east1
gcloud config set run/region $REGION

gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com secretmanager.googleapis.com
```

---

## 1️⃣ 建立最小應用（v1）

```bash
mkdir -p ~/pcd-lab01 && cd ~/pcd-lab01
```

`main.py`
```python
import os, json, signal, sys, time
from flask import Flask, jsonify

app = Flask(__name__)
VERSION = os.environ.get("APP_VERSION", "v1")
START = time.time()

def log(severity, message, **kw):
    print(json.dumps({"severity": severity, "message": message, "version": VERSION, **kw}), flush=True)

@app.get("/")
def root():
    log("INFO", "request received", path="/")
    return jsonify(version=VERSION, uptime_s=round(time.time() - START, 1))

@app.get("/healthz")
def healthz():
    return "ok", 200

@app.get("/slow")
def slow():                       # 用來觀察 concurrency 與擴充
    time.sleep(2)
    return jsonify(version=VERSION)

def graceful(signum, frame):
    log("NOTICE", "SIGTERM received, shutting down gracefully")
    sys.exit(0)
signal.signal(signal.SIGTERM, graceful)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

`requirements.txt`
```
Flask==3.0.3
gunicorn==22.0.0
```

`Procfile`（讓 Buildpacks 知道怎麼啟動）
```
web: gunicorn -b :$PORT -w 1 --threads 8 main:app
```

---

## 2️⃣ 建立專屬服務帳戶（最小權限）

```bash
gcloud iam service-accounts create lab01-sa --display-name="lab01 runtime"
SA=lab01-sa@$PROJECT.iam.gserviceaccount.com

# 只給觀測性需要的權限（不要用 default compute SA！）
for R in roles/logging.logWriter roles/cloudtrace.agent roles/monitoring.metricWriter; do
  gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$SA" --role="$R" --quiet
done
```
> 對應 [[IAM 與服務帳戶]] 的「專屬 SA + predefined role」原則。

---

## 3️⃣ 從原始碼部署（Buildpacks，無 Dockerfile）

```bash
gcloud run deploy lab01 \
  --source . \
  --service-account $SA \
  --set-env-vars APP_VERSION=v1 \
  --allow-unauthenticated \
  --concurrency 80 --cpu 1 --memory 512Mi \
  --min-instances 0 --max-instances 5 \
  --timeout 60

URL=$(gcloud run services describe lab01 --format='value(status.url)')
curl -s $URL | jq .
```

> [!question] 觀察 ①：冷啟動
> ```bash
> # 等 5 分鐘讓實例縮到 0（或直接觀察第一次呼叫）
> time curl -s $URL > /dev/null      # 第一次（冷）
> time curl -s $URL > /dev/null      # 第二次（暖）
> ```
> 記下兩者差異。這就是 `--min-instances` 要解決的問題。

---

## 4️⃣ 觀察並行與擴充

```bash
# 用 /slow（每個請求 2 秒）打 100 個併發
seq 100 | xargs -P 100 -I{} curl -s -o /dev/null $URL/slow
# 另一個終端觀察實例數
gcloud monitoring time-series list \
  --filter='metric.type="run.googleapis.com/container/instance_count" AND resource.labels.service_name="lab01"' \
  --format="value(points[0].value.int64Value)" 2>/dev/null | head
```

**改成 concurrency=1 再測一次**，比較實例數：
```bash
gcloud run services update lab01 --concurrency 1
# 再跑一次上面的壓測 → 實例數會明顯變多（也會更快撞到 max-instances=5）
gcloud run services update lab01 --concurrency 80    # 改回來
```
> [!success] 學到什麼
> `實例數 ≈ 進行中的請求數 ÷ concurrency`。這直接決定成本，見 [[成本與資源最佳化]]。

---

## 5️⃣ Canary 發布（**本 Lab 的重點**）

```bash
# 修改成 v2（改一行也好，這裡用環境變數模擬）
gcloud run deploy lab01 --source . --set-env-vars APP_VERSION=v2 \
  --no-traffic --tag candidate --service-account $SA

# 取得 candidate 的專屬 URL → 可以單獨測試，不影響任何使用者
CAND=$(gcloud run services describe lab01 --format='value(status.traffic[].url)' | tr ' ' '\n' | grep candidate)
curl -s $CAND | jq .          # 應該回 v2
curl -s $URL  | jq .          # 主 URL 仍然是 v1 ✅

# 10% canary
gcloud run services update-traffic lab01 --to-tags candidate=10
for i in $(seq 20); do curl -s $URL | jq -r .version; done | sort | uniq -c

# 50% → 100%
gcloud run services update-traffic lab01 --to-tags candidate=50
gcloud run services update-traffic lab01 --to-latest
```

**秒級回滾**
```bash
gcloud run revisions list --service lab01
PREV=$(gcloud run revisions list --service lab01 --format='value(metadata.name)' | tail -1)
time gcloud run services update-traffic lab01 --to-revisions $PREV=100
curl -s $URL | jq .           # 回到 v1
```
> [!success] 學到什麼
> 回滾不需要重新建置或部署 —— **只是改流量指向**。這是 revision 不可變帶來的好處。
> 對應 [[Cloud Deploy 與部署策略]]。

---

## 6️⃣ 加上驗證（服務對服務）

```bash
# 把服務改成需要驗證
gcloud run services update lab01 --no-allow-unauthenticated

curl -s -o /dev/null -w "%{http_code}\n" $URL            # 403
curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" $URL | jq .   # 200 ✅

# 建立一個呼叫者 SA，只給 invoker 角色
gcloud iam service-accounts create lab01-caller
CALLER=lab01-caller@$PROJECT.iam.gserviceaccount.com
gcloud run services add-iam-policy-binding lab01 \
  --member="serviceAccount:$CALLER" --role="roles/run.invoker"
```
> [!question] 觀察 ②
> 為什麼 `gcloud auth print-identity-token`（ID token）可以，`print-access-token` 不行？
> 答案在 [[驗證與授權 ADC OAuth JWT]]。

---

## 7️⃣ 掛上祕密與最小實例

```bash
echo -n "super-secret-value" | gcloud secrets create lab01-secret --data-file=-
gcloud secrets add-iam-policy-binding lab01-secret \
  --member="serviceAccount:$SA" --role="roles/secretmanager.secretAccessor"

gcloud run services update lab01 \
  --set-secrets "APP_SECRET=lab01-secret:latest" \
  --min-instances 1                      # 消除冷啟動（會持續計費！）

time curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token)" $URL > /dev/null
```
> 記得 lab 結束把 `--min-instances` 改回 0，否則持續計費。

---

## 8️⃣ 看 log（結構化）

```bash
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="lab01" AND jsonPayload.version="v2"' \
  --limit 5 --format=json | jq '.[].jsonPayload'
```
> 注意 `severity` 與自訂欄位 `version` 都能被查詢 → [[Cloud Logging]] 的結構化記錄。

---

## ✅ 檢核清單

- [ ] 能說出 Service / Revision / Instance 的差別
- [ ] 量測到冷啟動與暖啟動的差異
- [ ] 觀察到 concurrency 改變後實例數的變化
- [ ] 成功用 tag URL 測試新版本而**不影響主 URL**
- [ ] 完成 10% → 50% → 100% 的 canary
- [ ] 完成秒級回滾並確認版本回到 v1
- [ ] 理解 ID token 與 access token 的差別（實測過 403 / 200）
- [ ] 用結構化欄位查到 log

## 🧹 清理

```bash
gcloud run services delete lab01 --quiet
gcloud secrets delete lab01-secret --quiet
gcloud iam service-accounts delete $SA --quiet
gcloud iam service-accounts delete $CALLER --quiet
gcloud artifacts repositories delete cloud-run-source-deploy --location=$REGION --quiet 2>/dev/null
```

## 🔗 相關

- [[Cloud Run]]
- [[Cloud Deploy 與部署策略]]
- [[IAM 與服務帳戶]]
- [[驗證與授權 ADC OAuth JWT]]
- [[Secret Manager 與 Cloud KMS]]
- [[Cloud Logging]]
- [[成本與資源最佳化]]
- [[Section 3 設定雲端原生應用的部署]]
- [[25 天衝刺計劃]]
