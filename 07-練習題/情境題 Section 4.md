---
title: 情境題 Section 4
tags:
  - gcp/pcd
  - practice
  - exam/s4
status: 未做
confidence: 1
題數: 17
updated: 2026-09-27
---

# 情境題：Section 4（整合 Google Cloud 服務）

> 使用方式同 [[情境題 Section 1]]。這一節的題目偏「程式碼與診斷」。

---

## 🔌 4.1 資料與儲存整合

### Q1
Cloud Run 服務在高流量時出現 `FATAL: sorry, too many clients already`（Cloud SQL 連線數上限）。目前每個實例的連線池 `pool_size=20`，`--max-instances=100`。

**A.** 提高 Cloud SQL 的機器規格
**B.** 把 `pool_size` 降到 2～5、限制 `--max-instances`、必要時加 read replica 或快取
**C.** 改用公開 IP 連線
**D.** 每個請求建立新連線後關閉

> [!success]- 答案與判準
> **B**
> **判準**：`20 × 100 = 2000` 條連線遠超上限 → 要從**兩端**壓低（小連線池 + 限制實例數）。
> 補充手段：read replica 分流讀取、[[Memorystore 與快取策略]] 降低查詢、PgBouncer。
> A 只是把上限往上推（治標且貴）；C 與連線數無關；D 反而更糟（握手成本 + 瞬間連線暴增）。
> → [[Cloud SQL 與 AlloyDB]]、[[Cloud API 呼叫最佳實務]]

---

### Q2
下列哪段 Cloud Run 程式碼有效能問題？

```python
# A
from google.cloud import firestore
db = firestore.Client()
def handler(req):
    return db.document(f"orders/{req.args['id']}").get().to_dict()

# B
from google.cloud import firestore
def handler(req):
    db = firestore.Client()
    return db.document(f"orders/{req.args['id']}").get().to_dict()
```

> [!success]- 答案與判準
> **B 有問題**
> **判準**：**用戶端必須在全域建立一次並重用**。每個請求 `firestore.Client()` = 重新做驗證、建立連線、TLS 握手 → 延遲增加、連線數暴增。
> Cloud Run 實例會被重複使用，所以全域初始化的成本只付一次。
> → [[Cloud API 呼叫最佳實務]]、[[程式碼片段 Python Node Go]]

---

### Q3
你要列出一個 bucket 裡 500 萬個物件的名稱並逐一處理。

**A.** `blobs = list(client.list_blobs(bucket))` 然後迭代
**B.** 直接用 iterator：`for blob in client.list_blobs(bucket, prefix="..."):`
**C.** 一次 `page_size=5000000`
**D.** 用 `gcloud storage ls` 的輸出存成檔案再讀

> [!success]- 答案與判準
> **B**
> **判準**：`list()` 會把全部載入記憶體 → OOM。**iterator 自動翻頁**、記憶體恆定。
> 必要時用 `.pages` 明確控制每頁大小，並用 `prefix` 縮小範圍。
> → [[Cloud API 呼叫最佳實務]]

---

### Q4
你的服務要把每秒 5 萬筆點擊事件寫進 BigQuery 做即時分析，且希望**程式碼最少**。

**A.** 每筆事件呼叫一次 `insertAll`
**B.** 事件發到 Pub/Sub，用 **BigQuery 訂閱**直接寫入
**C.** 每筆事件寫成一個 GCS 檔案再 load
**D.** 在應用裡直接用 SQL `INSERT`

> [!success]- 答案與判準
> **B**
> **判準**：`即時` + `程式碼最少` → **Pub/Sub BigQuery 訂閱**（完全不用寫消費者）。
> 若要自己寫，則用 **Storage Write API**（高吞吐、支援 exactly-once），不要用舊的 `insertAll`。
> C 會產生 5 萬個小檔案；D 的單筆 `INSERT` DML 在 BigQuery 上極慢且有配額限制。
> → [[BigQuery 給開發者]]、[[Pub Sub]]

---

### Q5
你的 Pub/Sub 消費者（Cloud Run push）會偶發地重複處理同一則訊息，造成重複寄送 Email。

**A.** 在訂閱上啟用 exactly-once delivery 就完全解決了
**B.** 用訊息的業務鍵（例如 `orderId`）當冪等鍵，在 Firestore 用 `create()` 或 Redis `SET NX` 做去重
**C.** 延長 ack deadline
**D.** 改用 pull 訂閱

> [!success]- 答案與判準
> **B**
> **判準**：Pub/Sub 是 **at-least-once**，重複是**規格不是 bug** → 消費端必須**冪等**。
> A 是誘答：exactly-once 只保證「ack 之後不再送達同一訂閱」，**若處理完成但 ack 前崩潰，重啟後仍會再處理一次** → 冪等仍然必要。
> C 可以減少（處理慢造成的）重送但不能消除；D 換傳遞方式不改變保證。
> → [[Pub Sub]]、[[韌性模式 重試 冪等 退避 斷路器]]

---

### Q6
一則 Pub/Sub 訊息因格式錯誤持續處理失敗，導致訂閱不斷重試、費用上升。

**A.** 刪除訂閱重建
**B.** 設定 **dead-letter topic** 與 `max-delivery-attempts`，並對 DLQ 的訊息數設警示
**C.** 在消費端對所有錯誤回 2xx
**D.** 縮短訊息保留期

> [!success]- 答案與判準
> **B**
> **判準**：`毒藥訊息` → **DLQ + 嘗試次數上限**。
> 額外重點：**DLQ 建了要有人看** → 對 DLQ 的訊息數/積壓設警示，否則問題只是被藏起來。
> C 會讓真正的暫時性錯誤也被吞掉（資料遺失）；A/D 不解決根因。
> → [[Pub Sub]]、[[Cloud Monitoring 與 SLO]]

---

## 🔗 4.2 使用 Cloud API

### Q7
呼叫 Google Cloud API 時收到 `429 RESOURCE_EXHAUSTED`。最正確的處理組合是？

**A.** 立即重試直到成功
**B.** 指數退避（含 jitter）+ 最大重試次數，同時降低併發（`--max-instances` / Cloud Tasks 節流），必要時申請提高配額
**C.** 改用 REST 而非 gRPC
**D.** 忽略錯誤並回傳空結果

> [!success]- 答案與判準
> **B**
> **判準**：`429` 是**可重試**錯誤，但要**退避 + jitter**，並從源頭降低呼叫量。
> A 會造成重試風暴（thundering herd）；C 無關；D 隱藏問題。
> 最簡單的正確實作：**用 Cloud Client Library 的內建重試**。
> → [[Cloud API 呼叫最佳實務]]、[[韌性模式 重試 冪等 退避 斷路器]]

---

### Q8
下列錯誤中，哪些**不應該**重試？（選三個）

**A.** `400 INVALID_ARGUMENT`　**B.** `403 PERMISSION_DENIED`　**C.** `429 RESOURCE_EXHAUSTED`
**D.** `404 NOT_FOUND`　**E.** `503 UNAVAILABLE`　**F.** `409 ABORTED`

> [!success]- 答案與判準
> **A、B、D**
> `400`（請求本身錯）、`403`（缺角色）、`404`（資源不存在）→ 重試永遠失敗。
> `429`/`503` 退避重試；`409 ABORTED`（交易衝突）→ **重試整個交易**。
> → [[Cloud API 呼叫最佳實務]]

---

### Q9
你的程式呼叫 Cloud Run 服務 B 時一直收到 `401 Unauthorized`。程式用的是 `gcloud auth print-access-token` 取得的 token。

**A.** 給呼叫方 `roles/run.admin`
**B.** 改用 **ID token**，並把 `audience` 設為服務 B 的 URL
**C.** 把服務 B 設為 `--allow-unauthenticated`
**D.** 重新產生服務帳戶金鑰

> [!success]- 答案與判準
> **B**
> **判準**：呼叫 **Cloud Run / Cloud Functions / IAP** 這類服務端點要用 **ID token**（`aud` = 目標 URL）；
> 呼叫 **Google Cloud API**（GCS、Pub/Sub…）才用 access token。
> 注意區分：**401 = 憑證問題；403 = 缺 IAM 角色**（若 token 對但沒有 `run.invoker` 才是 403）。
> → [[驗證與授權 ADC OAuth JWT]]

---

### Q10
你的服務每個 HTTP 請求都呼叫 `access_secret_version` 取得資料庫密碼，造成延遲上升並偶發 429。

**A.** 改用環境變數硬寫密碼
**B.** 在啟動時讀取一次（或加上快取），並考慮用 volume 掛載以支援輪替
**C.** 提高 Secret Manager 配額
**D.** 把密碼存進 ConfigMap

> [!success]- 答案與判準
> **B**
> **判準**：祕密**不該在請求路徑上讀取** → 啟動時讀一次或加快取。
> 若需要「輪替後不重新部署就生效」→ 用 **volume 掛載 `latest`**（env 注入只在啟動時解析）。
> A/D 都是把祕密明文化。
> → [[Secret Manager 與 Cloud KMS]]、[[Cloud API 呼叫最佳實務]]

---

### Q11
你要一次寫入 800 筆文件到 Firestore。

**A.** 用一個 `batch()` 一次送 800 筆
**B.** 拆成兩個 batch（例如 500 + 300），或用平行的個別寫入
**C.** 用交易包住 800 筆
**D.** 一筆一筆同步寫入

> [!success]- 答案與判準
> **B**
> **判準**：Firestore 的 **batched write / transaction 上限是 500 個操作**。
> A/C 會直接失敗；D 慢且浪費往返。
> → [[Firestore]]、[[數字與限制速記]]

---

### Q12
呼叫某個 API 時只需要 `name` 與 `size` 兩個欄位，但回應很大造成延遲。

**A.** 改用分頁
**B.** 使用 **field mask / partial response**（例如 `fields="items(name,size),nextPageToken"`）
**C.** 提高 timeout
**D.** 把回應存到 Memorystore

> [!success]- 答案與判準
> **B**
> **判準**：`reduce the amount of data returned` → **欄位遮罩**。
> A 減少每次的筆數但每筆仍是完整物件；D 是另一個議題（重複讀取才有用）。
> → [[Cloud API 呼叫最佳實務]]、[[API 設計 REST 與 gRPC]]

---

## 🔭 4.3 疑難排解與可觀測性

### Q13
一個請求經過 API Gateway → Cloud Run A → Pub/Sub → Cloud Run B。你想一次取得這條鏈路的所有 log。

**A.** 對每個服務分別查 log 再手動比對時間
**B.** 在服務間傳播 trace context（`traceparent`），並在每筆 log 寫入 `logging.googleapis.com/trace`，然後用 `trace="projects/P/traces/ID"` 查詢
**C.** 把所有服務的 log 匯到同一個 GCS bucket
**D.** 用 Cloud Profiler

> [!success]- 答案與判準
> **B**
> **判準**：**trace 關聯的三個條件**：① 傳播 trace context、② 從 header 繼承、③ **log 寫入 trace 欄位**。
> 跨 Pub/Sub 要把 context 放進**訊息屬性**。少了第 ③ 步，你有 trace 也無法一鍵跳到 log。
> → [[OpenTelemetry 與 Trace 關聯]]、[[Cloud Logging]]

---

### Q14
你的例外沒有出現在 Error Reporting 裡。最可能的原因是？（選兩個）

**A.** log 的 `severity` 是 `INFO`
**B.** stack trace 被拆成多行多筆 log
**C.** 沒有啟用 Cloud Trace
**D.** 沒有設定 uptime check

> [!success]- 答案與判準
> **A 與 B**
> Error Reporting 需要：**`severity >= ERROR`** 且 **完整 stack trace 在同一筆 log 的同一個字串欄位**。
> 另兩個常見原因：例外被 `except: pass` 吞掉、格式不像 stack trace。
> → [[Cloud Trace Profiler 與 Error Reporting]]、[[Cloud Logging]]

---

### Q15
p99 延遲從 200 ms 變成 2 s。你會依什麼順序使用工具？

**A.** Profiler → Logging → Monitoring → Trace
**B.** Monitoring（看 SLI 找範圍）→ Cloud Trace（找最慢的 span）→ Logging（用 trace ID 撈上下文）→ Profiler（若瓶頸在自己的程式碼）
**C.** 直接看 Error Reporting
**D.** 重啟所有服務

> [!success]- 答案與判準
> **B**
> **判準**：**metrics 告訴你「有問題」→ trace 告訴你「在哪裡」→ log 告訴你「為什麼」→ profiler 告訴你「哪一行」**。
> → [[Section 4 整合 Google Cloud 服務]]、[[Cloud Trace Profiler 與 Error Reporting]]

---

### Q16
你要對「失敗的付款」設警示，但不想改程式碼新增自訂指標。程式已經輸出 `{"severity":"ERROR","error_code":"CARD_DECLINED",...}`。

**A.** 用 OpenTelemetry 新增自訂指標
**B.** 建立 **log-based metric**（filter `jsonPayload.error_code="CARD_DECLINED"`），再對它設 alerting policy
**C.** 建立 uptime check
**D.** 用 Cloud Profiler

> [!success]- 答案與判準
> **B**
> **判準**：`已經有 log` + `不想改程式` → **log-based metric**（最快的路）。
> 若需要**精確計數、低延遲、或記錄業務數值** → 才用自訂指標（OTel）。
> → [[Cloud Logging]]、[[Cloud Monitoring 與 SLO]]

---

### Q17
團隊抱怨警示太多（一天數十個），大家都開始忽略通知。目前的警示是「錯誤率 > 1% 持續 5 分鐘」。

**A.** 把門檻提高到 5%
**B.** 建立 SLO（例如 99.9% / 30 天）並改用**多視窗 burn-rate 警示**（快速燒 page、慢速燒開 ticket）
**C.** 關掉警示
**D.** 只在上班時間發警示

> [!success]- 答案與判準
> **B**
> **判準**：`alert fatigue` → **SLO + burn rate**。
> 因為 burn rate 考慮的是「這對月度 error budget 的實際影響」，短暫的抖動不會觸發，真正吃掉預算的事件才會。
> A 只是換一個任意數字；C/D 是逃避。
> → [[Cloud Monitoring 與 SLO]]

---

## 📊 作答紀錄

| 日期 | 答對 / 17 | 錯題號 | 主要錯因 |
|---|---|---|---|
| | | | |

## 🔗 相關

- [[Section 4 整合 Google Cloud 服務]]
- [[情境題 Section 1]]
- [[情境題 Section 2 與 3]]
- [[常見陷阱與誘答選項識別]]
- [[Cloud API 呼叫最佳實務]]
- [[OpenTelemetry 與 Trace 關聯]]
