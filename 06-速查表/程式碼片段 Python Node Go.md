---
title: 程式碼片段 Python Node Go
tags:
  - gcp/pcd
  - cheatsheet
status: 未讀
confidence: 1
updated: 2026-09-27
---

# 程式碼片段：Python / Node.js / Go

> [!tip] 考試會考「程式碼哪裡錯」
> 重點模式：**全域重用用戶端、分頁、退避重試、冪等、結構化 log、ID token**。
> 把這幾個模式的「正確長相」記住，看到錯的就能認出來。

---

## 1️⃣ 用戶端初始化（全域重用）

```python
# Python ✅
from google.cloud import firestore, storage, pubsub_v1
db = firestore.Client()                      # 模組載入時建立一次
gcs = storage.Client()
publisher = pubsub_v1.PublisherClient()

def handler(request):
    return db.collection("orders").document(request.args["id"]).get().to_dict()
```
```javascript
// Node.js ✅
const {Firestore} = require('@google-cloud/firestore');
const {Storage} = require('@google-cloud/storage');
const db = new Firestore();                  // 模組層級
const gcs = new Storage();

exports.handler = async (req, res) => {
  const doc = await db.doc(`orders/${req.query.id}`).get();
  res.json(doc.data());
};
```
```go
// Go ✅
package main

import (
    "context"
    "log"
    "cloud.google.com/go/firestore"
)

var db *firestore.Client

func init() {
    var err error
    db, err = firestore.NewClient(context.Background(), projectID)   // 一次
    if err != nil { log.Fatal(err) }
}
```
> ❌ 反模式：在 handler 內 `firestore.Client()` / `new Firestore()` → 每個請求重新驗證與建立連線。

---

## 2️⃣ 分頁（不要一次全抓）

```python
# Python
for blob in gcs.list_blobs("my-bucket", prefix="logs/"):        # iterator 自動翻頁
    process(blob)

for page in gcs.list_blobs("my-bucket", page_size=1000).pages:  # 明確控制每頁
    for blob in page:
        process(blob)

# Firestore cursor 分頁（不要用 offset）
q = db.collection("orders").order_by("createdAt").limit(50)
snap = list(q.stream())
while snap:
    for d in snap: process(d)
    snap = list(q.start_after(snap[-1]).stream())
```
```javascript
// Node.js
const [files] = await gcs.bucket('my-bucket').getFiles({autoPaginate: false, maxResults: 1000});
// 或串流
gcs.bucket('my-bucket').getFilesStream().on('data', f => process(f));
```
```go
// Go
it := client.Bucket("my-bucket").Objects(ctx, &storage.Query{Prefix: "logs/"})
for {
    attrs, err := it.Next()
    if err == iterator.Done { break }
    if err != nil { return err }
    process(attrs)
}
```

---

## 3️⃣ 退避重試

```python
# Python：用程式庫內建（推薦）
from google.api_core import retry, exceptions
r = retry.Retry(
    predicate=retry.if_exception_type(
        exceptions.ServiceUnavailable, exceptions.TooManyRequests,
        exceptions.InternalServerError, exceptions.DeadlineExceeded),
    initial=0.5, maximum=32.0, multiplier=2.0, deadline=120.0)
blob.upload_from_string("data", retry=r, timeout=30)

# 手刻（含 jitter）
import random, time
def with_backoff(fn, attempts=5, base=0.5, cap=32.0):
    for i in range(attempts):
        try:
            return fn()
        except RetryableError:
            if i == attempts - 1: raise
            d = min(base * 2 ** i, cap)
            time.sleep(d + random.uniform(0, d * 0.5))     # ⭐ jitter
```
```javascript
// Node.js：程式庫內建
await bucket.file('o').save('data', {
  retryOptions: {autoRetry: true, maxRetries: 5, retryDelayMultiplier: 2, maxRetryDelay: 32},
});
```
```go
// Go：gax 退避
import "github.com/googleapis/gax-go/v2"
err := gax.Invoke(ctx, func(ctx context.Context, _ gax.CallSettings) error {
    return doWork(ctx)
}, gax.WithRetry(func() gax.Retryer {
    return gax.OnCodes([]codes.Code{codes.Unavailable, codes.ResourceExhausted},
        gax.Backoff{Initial: 500 * time.Millisecond, Max: 32 * time.Second, Multiplier: 2})
}))
```

---

## 4️⃣ 呼叫需驗證的 Cloud Run（ID token）

```python
# Python
import google.auth.transport.requests, google.oauth2.id_token, requests
AUD = "https://svc-b-xxxx.a.run.app"
tok = google.oauth2.id_token.fetch_id_token(google.auth.transport.requests.Request(), AUD)
resp = requests.get(f"{AUD}/api", headers={"Authorization": f"Bearer {tok}"}, timeout=10)
```
```javascript
// Node.js
const {GoogleAuth} = require('google-auth-library');
const auth = new GoogleAuth();
const AUD = 'https://svc-b-xxxx.a.run.app';
const client = await auth.getIdTokenClient(AUD);
const res = await client.request({url: `${AUD}/api`});
```
```go
// Go
import "google.golang.org/api/idtoken"
aud := "https://svc-b-xxxx.a.run.app"
client, err := idtoken.NewClient(ctx, aud)
resp, err := client.Get(aud + "/api")
```
> ⚠️ 用 **access token** 呼叫 Cloud Run 會得到 **401**。見 [[驗證與授權 ADC OAuth JWT]]。

---

## 5️⃣ 冪等消費者

```python
# Python：Firestore create() 當去重鎖
from google.cloud import firestore
from google.api_core import exceptions
db = firestore.Client()

def handle(event_id, payload):
    try:
        db.document(f"processed/{event_id}").create({"at": firestore.SERVER_TIMESTAMP})
    except exceptions.AlreadyExists:
        return "duplicate"
    do_work(payload)
```
```javascript
// Node.js：Redis SET NX
const ok = await redis.set(`processed:${eventId}`, '1', 'NX', 'EX', 86400);
if (!ok) return 'duplicate';
await doWork(payload);
```
```sql
-- SQL：唯一索引 + UPSERT
INSERT INTO payments (idempotency_key, order_id, amount) VALUES ($1,$2,$3)
ON CONFLICT (idempotency_key) DO NOTHING;
```

---

## 6️⃣ 結構化 log（帶 trace ID）

```python
# Python
import json, os, sys
from opentelemetry import trace
PROJECT = os.environ["GOOGLE_CLOUD_PROJECT"]

def log(severity, message, **fields):
    ctx = trace.get_current_span().get_span_context()
    entry = {"severity": severity, "message": message, **fields}
    if ctx and ctx.trace_id:
        entry["logging.googleapis.com/trace"] = f"projects/{PROJECT}/traces/{format(ctx.trace_id,'032x')}"
        entry["logging.googleapis.com/spanId"] = format(ctx.span_id, "016x")
    print(json.dumps(entry), file=sys.stdout, flush=True)

log("INFO", "order created", order_id="o-987", amount=299)
```
```javascript
// Node.js
function log(severity, message, fields = {}) {
  const entry = {severity, message, ...fields};
  const t = process.env.TRACE_ID;   // 或從 OTel context 取
  if (t) entry['logging.googleapis.com/trace'] = `projects/${process.env.GOOGLE_CLOUD_PROJECT}/traces/${t}`;
  console.log(JSON.stringify(entry));
}
```
```go
// Go
type entry struct {
    Severity string `json:"severity"`
    Message  string `json:"message"`
    Trace    string `json:"logging.googleapis.com/trace,omitempty"`
    OrderID  string `json:"order_id,omitempty"`
}
func logJSON(e entry) { b, _ := json.Marshal(e); fmt.Println(string(b)) }
```

---

## 7️⃣ Pub/Sub 發布與訂閱

```python
# 發布（含 ordering key 與批次）
from google.cloud import pubsub_v1
settings = pubsub_v1.types.BatchSettings(max_messages=1000, max_latency=0.05)
publisher = pubsub_v1.PublisherClient(
    batch_settings=settings,
    publisher_options=pubsub_v1.types.PublisherOptions(enable_message_ordering=True))
future = publisher.publish(topic, b'{"id":1}', ordering_key="user-123", tier="vip")
print(future.result())

# 訂閱（含 flow control）
subscriber = pubsub_v1.SubscriberClient()
def callback(message):
    try:
        handle(message.data)
        message.ack()
    except TransientError:
        message.nack()
flow = pubsub_v1.types.FlowControl(max_messages=100)
subscriber.subscribe(sub_path, callback=callback, flow_control=flow).result()
```
```javascript
// Node.js
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
await pubsub.topic('orders').publishMessage({json: {id: 1}, attributes: {tier: 'vip'}});

pubsub.subscription('orders-sub', {flowControl: {maxMessages: 100}})
  .on('message', async msg => { try { await handle(msg.data); msg.ack(); } catch { msg.nack(); } });
```

---

## 8️⃣ Cloud Storage：signed URL 與 resumable upload

```python
from datetime import timedelta
blob = gcs.bucket("uploads").blob(f"users/{uid}/photo.jpg")

read_url  = blob.generate_signed_url(version="v4", expiration=timedelta(minutes=15), method="GET")
write_url = blob.generate_signed_url(version="v4", expiration=timedelta(minutes=15),
                                     method="PUT", content_type="image/jpeg")
# 大檔案：resumable
blob.upload_from_filename("big.mp4", timeout=600)   # 程式庫自動用 resumable
```
```javascript
const [url] = await gcs.bucket('uploads').file('big.mp4')
  .getSignedUrl({version: 'v4', action: 'write', expires: Date.now() + 15*60*1000, contentType: 'video/mp4'});
```

---

## 9️⃣ Cloud SQL 連線（小連線池 + IAM 驗證）

```python
import sqlalchemy
from google.cloud.sql.connector import Connector
connector = Connector()

def getconn():
    return connector.connect("PROJECT:asia-east1:orders-db", "pg8000",
                             user="app-sa@PROJECT.iam", db="orders", enable_iam_auth=True)

engine = sqlalchemy.create_engine("postgresql+pg8000://", creator=getconn,
                                  pool_size=2, max_overflow=1, pool_recycle=1800, pool_pre_ping=True)
```
```javascript
const {Connector} = require('@google-cloud/cloud-sql-connector');
const connector = new Connector();
const opts = await connector.getOptions({instanceConnectionName: 'PROJECT:asia-east1:orders-db', authType: 'IAM'});
const pool = new Pool({...opts, user: 'app-sa@PROJECT.iam', database: 'orders', max: 2});
```

---

## 🔟 優雅關閉

```python
import signal, sys
def graceful(signum, frame):
    print('{"severity":"NOTICE","message":"SIGTERM received"}', flush=True)
    # 停止接受新工作、等進行中的完成
    sys.exit(0)
signal.signal(signal.SIGTERM, graceful)
```
```javascript
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));      // 排空進行中的請求
});
```
```go
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGTERM)
go func() {
    <-c
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}()
```

---

## 1️⃣1️⃣ Vertex AI / Gemini（2026 考點）

```python
from google import genai
from google.genai import types
client = genai.Client(vertexai=True, project=PROJECT, location="asia-east1")

resp = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="摘要這段客服對話",
    config=types.GenerateContentConfig(
        temperature=0.2, max_output_tokens=512,
        response_mime_type="application/json"),     # ⭐ 結構化輸出
)

# 串流
for chunk in client.models.generate_content_stream(model="gemini-2.5-flash", contents=prompt):
    yield chunk.text
```

## 🔗 相關

- [[Cloud API 呼叫最佳實務]]
- [[韌性模式 重試 冪等 退避 斷路器]]
- [[驗證與授權 ADC OAuth JWT]]
- [[Cloud Logging]]
- [[OpenTelemetry 與 Trace 關聯]]
- [[Pub Sub]]
- [[Cloud Storage]]
- [[Cloud SQL 與 AlloyDB]]
- [[Vertex AI Gemini API 給開發者]]
- [[gcloud 速查]]
