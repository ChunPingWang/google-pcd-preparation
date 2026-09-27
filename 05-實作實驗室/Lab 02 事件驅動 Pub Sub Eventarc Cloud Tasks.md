---
title: Lab 02 事件驅動 Pub Sub Eventarc Cloud Tasks
tags:
  - gcp/pcd
  - lab
  - service/pubsub
  - service/eventarc
  - service/cloud-tasks
status: 未做
confidence: 1
預估時間: 75 分鐘
updated: 2026-09-27
---

# Lab 02：事件驅動三本柱（Pub/Sub + Eventarc + Cloud Tasks）

> [!abstract] 你會學到
> **親手體驗重複送達**、實作冪等消費、設定 dead-letter topic、用 Eventarc 接 GCS 事件、用 Cloud Tasks 控速與延遲執行。
> 對應筆記：[[Pub Sub]]、[[Eventarc]]、[[Cloud Tasks]]、[[韌性模式 重試 冪等 退避 斷路器]]、[[決策樹 訊息與事件選型]]

---

## 0️⃣ 準備

```bash
export PROJECT=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT --format='value(projectNumber)')
export REGION=asia-east1
gcloud config set run/region $REGION

gcloud services enable run.googleapis.com pubsub.googleapis.com eventarc.googleapis.com \
  cloudtasks.googleapis.com firestore.googleapis.com storage.googleapis.com cloudbuild.googleapis.com

# Firestore（用來做冪等去重表）
gcloud firestore databases create --location=$REGION --type=firestore-native 2>/dev/null || true
```

---

## 1️⃣ 冪等的事件處理器

`main.py`
```python
import json, os, sys, base64, random
from flask import Flask, request
from google.cloud import firestore
from google.api_core import exceptions

app = Flask(__name__)
db = firestore.Client()                       # ⭐ 全域建立一次
FAIL_RATE = float(os.environ.get("FAIL_RATE", "0"))

def log(sev, msg, **kw):
    print(json.dumps({"severity": sev, "message": msg, **kw}), flush=True)

def already_processed(event_id: str) -> bool:
    """用 create() 的『已存在就失敗』特性做去重"""
    try:
        db.document(f"processed/{event_id}").create({"at": firestore.SERVER_TIMESTAMP})
        return False
    except exceptions.AlreadyExists:
        return True

# ---------- Pub/Sub push 端點 ----------
@app.post("/pubsub")
def pubsub_push():
    envelope = request.get_json(silent=True) or {}
    msg = envelope.get("message", {})
    event_id = msg.get("messageId")
    data = base64.b64decode(msg.get("data", "")).decode() if msg.get("data") else ""

    if already_processed(event_id):
        log("INFO", "duplicate skipped", event_id=event_id)
        return "", 204                        # ack，不重做

    if random.random() < FAIL_RATE:           # 故意失敗，觀察重送與 DLQ
        log("ERROR", "simulated failure", event_id=event_id)
        db.document(f"processed/{event_id}").delete()   # 讓重試能再進來
        return "simulated failure", 500       # 非 2xx → Pub/Sub 重送

    log("INFO", "processed", event_id=event_id, data=data)
    return "", 204

# ---------- Eventarc (GCS) 端點 ----------
@app.post("/gcs")
def gcs_event():
    event_id = request.headers.get("ce-id")
    event_type = request.headers.get("ce-type")
    payload = request.get_json(silent=True) or {}
    if already_processed(f"gcs-{event_id}"):
        return "", 204
    log("INFO", "gcs event", event_id=event_id, type=event_type,
        bucket=payload.get("bucket"), name=payload.get("name"))
    return "", 204

# ---------- Cloud Tasks worker ----------
@app.post("/task")
def task_worker():
    body = request.get_json(silent=True) or {}
    task_name = request.headers.get("X-CloudTasks-TaskName")
    retry_count = request.headers.get("X-CloudTasks-TaskRetryCount")
    log("INFO", "task executed", task=task_name, retry=retry_count, payload=body)
    return "", 200

@app.get("/healthz")
def healthz():
    return "ok", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

`requirements.txt`
```
Flask==3.0.3
gunicorn==22.0.0
google-cloud-firestore==2.19.0
```
`Procfile`
```
web: gunicorn -b :$PORT -w 1 --threads 8 main:app
```

部署：
```bash
gcloud iam service-accounts create lab02-sa
SA=lab02-sa@$PROJECT.iam.gserviceaccount.com
for R in roles/datastore.user roles/logging.logWriter; do
  gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$SA" --role=$R --quiet
done

gcloud run deploy lab02 --source . --service-account $SA \
  --no-allow-unauthenticated --set-env-vars FAIL_RATE=0
URL=$(gcloud run services describe lab02 --format='value(status.url)')
```

---

## 2️⃣ Pub/Sub：push 訂閱 + dead-letter topic

```bash
gcloud pubsub topics create lab02-events
gcloud pubsub topics create lab02-dlq

# 給 Pub/Sub 的推送身分
gcloud iam service-accounts create lab02-pubsub-invoker
PSSA=lab02-pubsub-invoker@$PROJECT.iam.gserviceaccount.com
gcloud run services add-iam-policy-binding lab02 --member="serviceAccount:$PSSA" --role=roles/run.invoker

# Pub/Sub 服務代理需要能建立 token
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com" \
  --role=roles/iam.serviceAccountTokenCreator

gcloud pubsub subscriptions create lab02-sub \
  --topic=lab02-events \
  --push-endpoint="$URL/pubsub" \
  --push-auth-service-account=$PSSA \
  --ack-deadline=30 \
  --dead-letter-topic=lab02-dlq --max-delivery-attempts=5

# DLQ 也要有訂閱才能觀察
gcloud pubsub subscriptions create lab02-dlq-sub --topic=lab02-dlq
```

**測試正常路徑**
```bash
gcloud pubsub topics publish lab02-events --message='{"orderId":"o-1"}'
gcloud logging read 'resource.labels.service_name="lab02" AND jsonPayload.message="processed"' --limit 3 --format=json | jq '.[].jsonPayload'
```

**測試重試與 DLQ**
```bash
gcloud run services update lab02 --set-env-vars FAIL_RATE=1.0     # 100% 失敗
gcloud pubsub topics publish lab02-events --message='{"orderId":"poison"}'

# 等 1～2 分鐘（5 次嘗試用盡）後檢查 DLQ
sleep 120
gcloud pubsub subscriptions pull lab02-dlq-sub --auto-ack --limit=5
```
> [!success] 學到什麼
> 沒有 DLQ → 這則毒藥訊息會**無限重試**。`max-delivery-attempts` 是你的保險絲。

**測試冪等**
```bash
gcloud run services update lab02 --set-env-vars FAIL_RATE=0
# 手動用同一個 messageId 打兩次（模擬重複送達）
TOKEN=$(gcloud auth print-identity-token)
BODY='{"message":{"messageId":"dup-test-1","data":"'"$(echo -n '{"orderId":"o-dup"}' | base64)"'"}}'
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$URL/pubsub" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "$BODY"
curl -s -o /dev/null -w "%{http_code}\n" -X POST "$URL/pubsub" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "$BODY"
gcloud logging read 'jsonPayload.message="duplicate skipped"' --limit 2 --format=json | jq '.[].jsonPayload'
```

---

## 3️⃣ Eventarc：GCS 上傳觸發

```bash
BUCKET=gs://lab02-uploads-$PROJECT
gcloud storage buckets create $BUCKET --location=$REGION

# GCS 服務代理需要 publisher 權限
GCS_SA=$(gcloud storage service-agent --project=$PROJECT)
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$GCS_SA" --role=roles/pubsub.publisher

gcloud iam service-accounts create lab02-eventarc
EVSA=lab02-eventarc@$PROJECT.iam.gserviceaccount.com
gcloud run services add-iam-policy-binding lab02 --member="serviceAccount:$EVSA" --role=roles/run.invoker
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$EVSA" --role=roles/eventarc.eventReceiver

gcloud eventarc triggers create lab02-upload \
  --location=$REGION \
  --destination-run-service=lab02 --destination-run-path=/gcs --destination-run-region=$REGION \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=lab02-uploads-$PROJECT" \
  --service-account=$EVSA

# 觸發
echo "hello" > /tmp/lab02.txt && gcloud storage cp /tmp/lab02.txt $BUCKET/
sleep 15
gcloud logging read 'jsonPayload.message="gcs event"' --limit 3 --format=json | jq '.[].jsonPayload'
```
> [!question] 觀察
> 若處理器把結果寫回**同一個 bucket**會發生什麼？（答案：無限迴圈 → 見 [[Eventarc]] 的陷阱）

---

## 4️⃣ Cloud Tasks：控速 + 延遲執行

```bash
gcloud tasks queues create lab02-queue --location=$REGION \
  --max-dispatches-per-second=2 --max-concurrent-dispatches=1 --max-attempts=3

gcloud iam service-accounts create lab02-tasks
TSA=lab02-tasks@$PROJECT.iam.gserviceaccount.com
gcloud run services add-iam-policy-binding lab02 --member="serviceAccount:$TSA" --role=roles/run.invoker
```

`enqueue.py`
```python
import datetime, json, os, sys
from google.cloud import tasks_v2
from google.protobuf import timestamp_pb2

PROJECT, REGION, URL, SA = sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4]
client = tasks_v2.CloudTasksClient()
parent = client.queue_path(PROJECT, REGION, "lab02-queue")

for i in range(10):
    task = {
        "http_request": {
            "http_method": tasks_v2.HttpMethod.POST,
            "url": f"{URL}/task",
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"i": i}).encode(),
            "oidc_token": {"service_account_email": SA, "audience": URL},
        }
    }
    if i == 9:      # 最後一個延遲 60 秒執行
        ts = timestamp_pb2.Timestamp()
        ts.FromDatetime(datetime.datetime.utcnow() + datetime.timedelta(seconds=60))
        task["schedule_time"] = ts
    client.create_task(parent=parent, task=task)
print("enqueued 10 tasks")
```

```bash
pip install -q google-cloud-tasks
python enqueue.py $PROJECT $REGION $URL $TSA

# 觀察：因為速率 2/s、併發 1，任務會被均勻派送（不是一次全打）
gcloud logging read 'jsonPayload.message="task executed"' --limit 15 \
  --format="table(timestamp, jsonPayload.payload.i)" --freshness=10m
```
> [!success] 學到什麼
> 10 個任務不是同時打過去 → **佇列層級的速率限制**就是保護下游的工具。
> 這是 Cloud Tasks 與 Pub/Sub 最大的差別（見 [[決策樹 訊息與事件選型]]）。

---

## ✅ 檢核清單

- [ ] 用 `create()` 實作出冪等去重，並實測「重複送達只處理一次」
- [ ] 觀察到毒藥訊息在 5 次嘗試後進入 **DLQ**
- [ ] 能說明沒有 DLQ 會發生什麼
- [ ] Eventarc 觸發成功，且能說出需要的三組 IAM 授權
- [ ] Cloud Tasks 的速率限制實際生效（時間戳分散）
- [ ] 成功建立延遲 60 秒執行的任務
- [ ] 能說出「同一個 topic 要讓兩個服務都處理」該怎麼做

## 🧹 清理

```bash
gcloud eventarc triggers delete lab02-upload --location=$REGION --quiet
gcloud pubsub subscriptions delete lab02-sub lab02-dlq-sub --quiet
gcloud pubsub topics delete lab02-events lab02-dlq --quiet
gcloud tasks queues delete lab02-queue --location=$REGION --quiet
gcloud storage rm -r $BUCKET
gcloud run services delete lab02 --quiet
for s in lab02-sa lab02-pubsub-invoker lab02-eventarc lab02-tasks; do
  gcloud iam service-accounts delete $s@$PROJECT.iam.gserviceaccount.com --quiet
done
```

## 🔗 相關

- [[Pub Sub]]
- [[Eventarc]]
- [[Cloud Tasks]]
- [[韌性模式 重試 冪等 退避 斷路器]]
- [[決策樹 訊息與事件選型]]
- [[Firestore]]
- [[Cloud Run]]
- [[Workflows 與 Cloud Scheduler]]
- [[25 天衝刺計劃]]
