---
title: Cloud API 呼叫最佳實務
tags:
  - gcp/pcd
  - pattern
  - exam/s4
status: 未讀
confidence: 1
importance: 5
updated: 2026-09-27
---

# 呼叫 Google Cloud API 的最佳實務

> [!abstract] 為什麼這篇重要
> 官方 Section 4.2 把考點寫得非常具體，**五個子項目直接列出來**：
> **批次請求、限制回傳資料、分頁、快取結果、錯誤處理（指數退避）**。
> 這是整份考試指南中最「照著背就能拿分」的一段。

---

## 🧰 第一原則：用 Cloud Client Libraries

| 選項 | 何時用 |
|---|---|
| **Cloud Client Libraries**（`google-cloud-*`） | ⭐ **預設答案**：慣用語法、內建 ADC 驗證、**內建退避重試**、自動分頁、連線池 |
| Google API Client Libraries（舊的 `googleapiclient`） | 沒有 Cloud Client Library 的服務 |
| **gRPC** 直接呼叫 | 需要極致效能或特殊控制 |
| **REST** 直接呼叫 | 沒有程式庫的環境、簡單腳本、webhook |
| **API Explorer / `gcloud`** | 探索與測試 |

> [!important] 考題判準
> 「呼叫 Cloud API 的建議方式」→ **Cloud Client Libraries**。
> 理由要能說出來：**驗證（ADC）、重試、分頁、慣用語法都已內建** → 少寫錯少踩坑。

### 啟用服務（容易漏）
```bash
gcloud services enable run.googleapis.com pubsub.googleapis.com firestore.googleapis.com
gcloud services list --enabled
```
沒啟用會得到 **`403 SERVICE_DISABLED`**（不是權限問題，別誤判去改 IAM）。

---

## 1️⃣ 用戶端重用（最常見的效能錯誤）

```python
# ✅ 全域建立一次，跨請求重用（Cloud Run 實例生命週期內共用）
from google.cloud import firestore, storage, pubsub_v1
db = firestore.Client()
gcs = storage.Client()
publisher = pubsub_v1.PublisherClient()

def handler(request):
    return db.collection("orders").document(request.args["id"]).get().to_dict()

# ❌ 每次請求都建立 → 重新做驗證、建立連線、TLS 握手
def bad(request):
    return firestore.Client().collection("orders")...
```
> 影響：延遲增加數十到數百毫秒、連線數爆炸、可能觸發配額。
> 這在 Cloud Run / Cloud Run functions 上尤其重要（實例會被重複使用）。

---

## 2️⃣ 分頁（Pagination）

```python
# ✅ 用 iterator，程式庫自動翻頁，記憶體恆定
for blob in gcs.list_blobs("my-bucket", prefix="logs/"):
    process(blob)

# ✅ 明確控制每頁大小（減少往返或減少單次負載）
for page in gcs.list_blobs("my-bucket", page_size=1000).pages:
    for blob in page:
        process(blob)

# ❌ 一次載入全部 → 500 萬個物件會 OOM
all_blobs = list(gcs.list_blobs("my-bucket"))
```
> **原則**：任何 `List*` API 都要假設結果可能很大 → 用 iterator / `pageToken`，**絕不 `list()` 全抓**。
> 對外提供 API 時也要做分頁 → 見 [[API 設計 REST 與 gRPC]]。

---

## 3️⃣ 限制回傳資料（Field mask / 部分回應）

```python
# 只取需要的欄位 → 省頻寬、省延遲、省序列化成本
blobs = gcs.list_blobs("my-bucket", fields="items(name,size),nextPageToken")
```
```python
# Firestore：只取部分欄位
doc = db.document("users/u1").get(field_paths=["name", "email"])
```
```bash
# gcloud 也支援
gcloud run services list --format="table(metadata.name, status.url)"
gcloud compute instances list --format="value(name)"
```
> 考點關鍵詞：`reduce the amount of data returned`、`improve performance` → **field mask / partial response**。

---

## 4️⃣ 批次請求（Batching）

| 服務 | 批次方式 |
|---|---|
| **Firestore** | `batch()`（最多 500 個寫入操作）、`get_all()` 批次讀 |
| **Pub/Sub** | publisher 的 **batch settings**（`max_messages`、`max_bytes`、`max_latency`）自動批次 |
| **BigQuery** | 批次載入 job（免費）；Storage Write API 的 append 批次 |
| **Cloud Storage** | 沒有真正的批次 API → 用**平行上傳/下載**（多執行緒）或 compose |
| **Monitoring** | `timeSeries.create` 一次送多筆時間序列 |
| REST 通用 | 部分 API 支援 `/batch` 端點（多個子請求包成一個 HTTP 請求） |

```python
# Pub/Sub 批次設定：延遲換吞吐
from google.cloud import pubsub_v1
settings = pubsub_v1.types.BatchSettings(max_messages=1000, max_bytes=1024*1024, max_latency=0.05)
publisher = pubsub_v1.PublisherClient(batch_settings=settings)
```
```python
# Firestore 批次寫入
batch = db.batch()
for item in items[:500]:                 # ⚠️ 上限 500
    batch.set(db.document(f"items/{item['id']}"), item)
batch.commit()
```
> [!warning] 批次的取捨
> 批次**增加延遲**（要等湊滿或等時限）但**提升吞吐**、降低成本。
> 即時性要求高的路徑不要批次；背景寫入則應該批次。
> 另外注意：**批次內單筆失敗的處理**（部分成功要能辨識哪幾筆失敗）。

---

## 5️⃣ 快取結果

| 什麼值得快取 | 放哪 |
|---|---|
| 很少變的中繼資料（設定、對照表、方案） | [[Memorystore 與快取策略]] 或程序內 |
| 昂貴的查詢結果 | Memorystore（設 TTL） |
| 祕密（Secret Manager 的值） | **啟動時讀一次或加快取**（不要每個請求呼叫） |
| 公開的靜態內容 | **Cloud CDN** |
| Token | 用戶端程式庫**已經自動快取**，不要自己管 |

> [!important] 三個必記的快取點
> 1. **祕密**：每個請求呼叫 Secret Manager = 延遲 + 配額風險。
> 2. **Token**：程式庫會快取並在到期前更新 → 不要自己呼叫 metadata server。
> 3. **設定/對照表**：用 TTL 快取，避免對資料庫的重複讀取。

---

## 6️⃣ 錯誤處理（官方明文考點）

### 錯誤碼 → 行動 對照表（**必背**）
| 錯誤 | gRPC 名稱 | 重試？ | 行動 |
|---|---|---|---|
| 400 | `INVALID_ARGUMENT` | ❌ | 修請求 |
| 401 | `UNAUTHENTICATED` | ❌ | 檢查 ADC / token 型別 |
| 403 | `PERMISSION_DENIED` | ❌ | **補 IAM 角色** |
| 403 | `SERVICE_DISABLED` | ❌ | `gcloud services enable` |
| 404 | `NOT_FOUND` | ❌ | 資源不存在 |
| 409 | `ALREADY_EXISTS` | ❌ | 通常代表「已處理過」→ 冪等成功 |
| 409 | `ABORTED` | ✅ | 交易衝突 → 重試整個交易 |
| 429 | `RESOURCE_EXHAUSTED` | ✅ **退避** | 超配額 → 退避 + 申請提額 |
| 499 | `CANCELLED` | ❌ | 客戶端取消 / timeout 太短 |
| 500 | `INTERNAL` | ✅ 退避 | 暫時性 |
| 503 | `UNAVAILABLE` | ✅ 退避 | 暫時性 |
| 504 | `DEADLINE_EXCEEDED` | ⚠️ 退避（**需冪等**） | 可能已執行成功 |

### 用程式庫的內建重試（推薦）
```python
from google.api_core import retry, exceptions
from google.cloud import storage

custom_retry = retry.Retry(
    predicate=retry.if_exception_type(
        exceptions.ServiceUnavailable, exceptions.TooManyRequests,
        exceptions.InternalServerError, exceptions.DeadlineExceeded),
    initial=0.5, maximum=32.0, multiplier=2.0, deadline=120.0,   # 含 jitter
)
blob = storage.Client().bucket("b").blob("o")
blob.upload_from_string("data", retry=custom_retry, timeout=30)
```
> 手刻退避的公式與 jitter 見 [[韌性模式 重試 冪等 退避 斷路器]]。

---

## 7️⃣ 配額與速率限制

| 概念 | 說明 |
|---|---|
| **配額類型** | 速率配額（每分鐘請求數）、配置配額（資源數量上限） |
| 查看/調整 | `IAM & Admin → Quotas`；可申請提高 |
| 監控 | 對 `serviceruntime.googleapis.com/quota/allocation/usage` 等指標設警示 |
| 自我保護 | 用 [[Cloud Tasks]] 節流、`--max-instances` 限制併發、應用內 rate limit |

> 考點：「呼叫 API 常遇到 429」→ **退避重試 + 降低併發（max-instances / Tasks 節流）+ 申請提額 + 批次減少呼叫數**。

---

## 🎯 考點速記

| 看到題目說… | 就想到 |
|---|---|
| `recommended way to call Google Cloud APIs` | **Cloud Client Libraries** |
| `reduce latency of repeated calls` | **重用用戶端** + 快取 |
| `list millions of objects without OOM` | **分頁 iterator** |
| `reduce response size / bandwidth` | **field mask / partial response** |
| `reduce the number of API calls` | **批次** + 快取 |
| `handle 429 / quota exceeded` | **指數退避 + jitter**（+ 降併發 + 提額） |
| `403 PERMISSION_DENIED` | 補 IAM 角色（不是換 token） |
| `403 SERVICE_DISABLED` | **啟用 API** |
| `ABORTED from a transaction` | 重試整個交易 |
| `secret lookup is slow` | 啟動時讀 / 快取 |
| `should we cache access tokens?` | **程式庫已處理**，不要自己管 |
| `batch has higher throughput but...` | **延遲增加**（取捨） |

## 💣 真實場景陷阱

1. **每請求建立用戶端**：最常見的效能問題。
2. **`list()` 全抓**：OOM 或超時。
3. **自己手刻重試沒有 jitter**：重試風暴。
4. **重試 `400`/`403`**：浪費配額且永不成功。
5. **在請求路徑上讀祕密**：延遲與配額雙殺。
6. **批次太大**：Firestore 超過 500 直接報錯；Pub/Sub 超過 10 MB 失敗。
7. **忽略部分成功**：批次中有幾筆失敗但程式當成全部成功。
8. **timeout 沒設**：呼叫卡住直到請求逾時。

## ✍️ 自我檢核

1. 官方列出的五個 API 呼叫最佳實務是什麼？
2. 為什麼要重用用戶端？在 Cloud Run 上放哪裡？
3. 列出五個錯誤碼與對應行動（哪些該重試）？
4. `403 PERMISSION_DENIED` 與 `403 SERVICE_DISABLED` 的處理差在哪？
5. 批次的好處與代價？Firestore 與 Pub/Sub 的批次上限？
6. 哪三種東西一定要快取？哪一種不需要自己快取？
7. 常遇到 429，四個方向的解法是什麼？

## 🔗 相關

- [[韌性模式 重試 冪等 退避 斷路器]]
- [[API 設計 REST 與 gRPC]]
- [[驗證與授權 ADC OAuth JWT]]
- [[Memorystore 與快取策略]]
- [[Secret Manager 與 Cloud KMS]]
- [[Cloud Tasks]]
- [[Pub Sub]]
- [[Firestore]]
- [[程式碼片段 Python Node Go]]
- [[Section 4 整合 Google Cloud 服務]]
