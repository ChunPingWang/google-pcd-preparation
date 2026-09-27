---
title: API 設計 REST 與 gRPC
tags:
  - gcp/pcd
  - pattern
  - exam/s1
status: 未讀
confidence: 1
importance: 3
updated: 2026-09-27
---

# API 設計：REST 與 gRPC

> [!abstract] 一句話定位
> 官方考點原文：**「建立與部署 API（例如 HTTP REST、gRPC）」**、**「版本化、公開與保護 API」**。
> 考的是**選型判準 + Google API 設計慣例 + 相容性規則**。

---

## 📘 技術理解
*原理、限制與實務操作 —— 不為考試也該懂的部分。*

### ⚖️ REST vs gRPC（**核心對照**）

| 維度 | **REST / JSON** | **gRPC** |
|---|---|---|
| 傳輸 | HTTP/1.1 或 HTTP/2 | **HTTP/2**（必須） |
| 序列化 | JSON（文字，人類可讀） | **Protocol Buffers**（二進位，體積小、快） |
| 契約 | OpenAPI（可選） | **`.proto` 強制契約** + 自動產生 client/server 程式碼 |
| 串流 | 有限（SSE / WebSocket） | **原生雙向串流** ⭐ |
| 瀏覽器支援 | ✅ 原生 | ⚠️ 需要 **gRPC-Web** + proxy |
| 效能 | 一般 | **明顯更好**（尤其高頻小請求） |
| 除錯 | `curl` 就能測 | 需要 `grpcurl` 等工具 |
| 適合 | **公開 API、瀏覽器、第三方整合** | **內部微服務間、高頻低延遲、串流** |

> [!important] 考題判準
> - `public API`、`third-party developers`、`browser clients`、`webhooks` → **REST**
> - `internal microservices`、`low latency`、`high throughput`、`bidirectional streaming`、`strongly typed contract` → **gRPC**
> - 兩者都要？→ 用 **gRPC + gRPC transcoding**（同一個服務同時提供 gRPC 與 REST/JSON 介面，Google 自家 API 就是這樣做的）

**在 Google Cloud 上部署 gRPC**
- **Cloud Run** 支援 gRPC（包含串流；unary 與 server streaming 都可以）。
- **GKE** 完全支援；搭配 **Cloud Service Mesh** 可做負載平衡與重試（gRPC 的長連線需要 L7 感知的 LB 才能均衡）。
- **健康檢查**：gRPC 服務要實作 **gRPC Health Checking Protocol**，或用 `grpc` probe。

---

### 📐 Google API 設計慣例（**資源導向**）

官方《API Design Guide》的核心：**API 是一組資源 + 標準方法**。

| 標準方法 | HTTP | 說明 |
|---|---|---|
| `List` | `GET /v1/orders` | 列出集合（**要分頁**） |
| `Get` | `GET /v1/orders/{id}` | 取單一資源 |
| `Create` | `POST /v1/orders` | 建立（可帶 `idempotency key`） |
| `Update` | `PATCH /v1/orders/{id}` | 部分更新（用 **field mask**） |
| `Delete` | `DELETE /v1/orders/{id}` | 刪除 |
| 自訂方法 | `POST /v1/orders/{id}:cancel` | 動詞用 `:verb` 形式 |

**資源命名**
```
/v1/customers/{customer}/orders/{order}/items/{item}
  ↑ 版本   ↑ 複數名詞（集合）    ↑ 資源 ID
```
- 集合用**複數名詞**，不要用動詞（`/getOrders` ❌）。
- 階層表達歸屬關係。
- 欄位用 `snake_case`（proto）或 `camelCase`（JSON），保持一致。

#### 分頁（必考）
```
GET /v1/orders?page_size=50&page_token=abc123
→ { "orders": [...], "next_page_token": "def456" }
```
> **用 token（cursor）分頁，不要用 offset**：offset 在大資料集上效率差且資料變動時會漏/重複。
> 相關：[[Cloud API 呼叫最佳實務]]

#### 部分更新與欄位遮罩
```
PATCH /v1/orders/123?update_mask=status,note
{ "status": "SHIPPED", "note": "expedited" }
```
> `update_mask` 明確指出「只改這些欄位」→ 避免把沒傳的欄位清空。
> 讀取端的對應是 `fields` / `FieldMask`（只回傳需要的欄位，省頻寬）。

#### 長時間執行的操作（LRO）
```
POST /v1/exports  →  202 { "name": "operations/abc", "done": false }
GET /v1/operations/abc  →  { "done": true, "response": {...} }
```
> 超過幾秒的操作**不要同步等待**。回一個 operation，讓客戶端輪詢或用 webhook/Pub/Sub 通知。
> 相關：[[Cloud Tasks]]、[[Workflows 與 Cloud Scheduler]]

#### 錯誤回應
用標準錯誤碼 + 結構化錯誤內容：
```json
{
  "error": {
    "code": 400,
    "status": "INVALID_ARGUMENT",
    "message": "field 'amount' must be positive",
    "details": [{ "@type": "type.googleapis.com/google.rpc.BadRequest", "fieldViolations": [...] }]
  }
}
```
錯誤碼對照見 [[Cloud API 呼叫最佳實務]]。

---

### 🔢 版本化與相容性（**Section 3.1 考點**）

| 做法 | 範例 | 評價 |
|---|---|---|
| **URI 路徑版本** | `/v1/orders`、`/v2/orders` | ⭐ Google 自家 API 用這種，最清楚 |
| Header 版本 | `Accept: application/vnd.api.v2+json` | 乾淨但對使用者不友善 |
| 查詢參數 | `?version=2` | 不建議 |

#### 向後相容的變更規則
| ✅ 可以（不破壞相容） | ❌ 不可以（破壞相容） |
|---|---|
| 新增**可選**欄位 | 刪除或重新命名欄位 |
| 新增端點 / 新增自訂方法 | 改變欄位型別或語意 |
| 新增 enum 值（客戶端要容忍未知值） | 把可選欄位變必填 |
| 放寬驗證規則 | 收緊驗證規則 |
| 新增可選的查詢參數 | 改變預設值或預設排序 |
| — | 改變錯誤碼的語意 |

> [!important] 破壞性變更的正確流程
> 1. 推出 **`/v2`**，`/v1` 繼續運作
> 2. 公告**棄用（deprecation）** 與 **sunset 日期**
> 3. 監控 v1 的使用量（用 [[Cloud Monitoring 與 SLO]] / [[API 管理 Apigee 與 API Gateway]] 的分析）
> 4. 到期後下線 v1
>
> 考題關鍵詞：`without breaking existing clients` → 新版本並行，不是直接改。

---

### 🔐 保護 API

| 需求 | 機制 |
|---|---|
| 識別呼叫的應用程式 | **API key**（不是驗證！） |
| 驗證終端使用者 | **JWT**（Identity Platform / Firebase / 第三方 IdP） |
| 驗證服務身分 | **Google ID token** + `roles/run.invoker` |
| 第三方代表使用者存取 | **OAuth 2.0 authorization code** |
| 限流 | **Apigee Quota/SpikeArrest**、API Gateway 配額、Cloud Armor、應用內 Redis |
| WAF / DDoS | **Cloud Armor** |
| 企業內部應用 | **IAP** |
| 服務間 mTLS | **Cloud Service Mesh** |

詳見 [[API 管理 Apigee 與 API Gateway]]、[[驗證與授權 ADC OAuth JWT]]。

---

### 🧱 其他實務要點

| 要點 | 說明 |
|---|---|
| **冪等性** | `PUT`/`DELETE` 天然冪等；`POST` 要靠 **`Idempotency-Key` header** → 見 [[韌性模式 重試 冪等 退避 斷路器]] |
| **限制回應大小** | 一律分頁；提供 `fields` 讓客戶端選欄位 |
| **壓縮** | `Accept-Encoding: gzip`（LB 可代為壓縮） |
| **快取** | 對 GET 設 `Cache-Control` / `ETag` → 可被 Cloud CDN 利用 |
| **批次** | 提供 `batchGet` / `batchCreate` 減少往返 |
| **HATEOAS 不是必須** | Google 的設計指南不強調它 |
| **契約先行** | 先定 `.proto` 或 OpenAPI，再實作；可產生 SDK 與 mock |

---

## 🎯 應試
*考場上的提取線索與自我測驗 —— 備考期才需要。*

### 🎯 考點速記

| 看到題目說… | 就想到 |
|---|---|
| `public API for third-party developers` | **REST + OpenAPI** |
| `internal service-to-service, low latency` | **gRPC** |
| `bidirectional streaming` | **gRPC** |
| `browser must call it directly` | REST（或 gRPC-Web） |
| `expose both gRPC and REST from one service` | **gRPC transcoding** |
| `list a very large collection` | **cursor/token 分頁** |
| `update only one field` | **PATCH + update_mask** |
| `operation takes 10 minutes` | **LRO（202 + operation）**，不要同步等 |
| `avoid breaking existing clients` | **新版本 `/v2` 並行 + 棄用流程** |
| `retry a POST safely` | **Idempotency-Key** |
| `reduce response payload size` | `fields` 欄位遮罩 + 分頁 + gzip |
| `gRPC 在 GKE 負載不均` | gRPC 長連線 → 需要 L7 LB / **Service Mesh** |

### 💣 真實場景陷阱

1. **offset 分頁**：資料量大時效能崩壞、資料變動會漏筆。
2. **`PUT` 當部分更新**：沒傳的欄位被清空。用 `PATCH + update_mask`。
3. **同步等待長操作**：超過 timeout 就失敗且客戶端不知道狀態。
4. **直接改 v1 的行為**：客戶端大量壞掉。
5. **enum 新增值導致舊客戶端崩潰**：客戶端要**容忍未知 enum**（這是設計時就要說明的契約）。
6. **gRPC 走 L4 LB**：連線建立後就固定，負載不均。
7. **API key 當驗證**：可被複製盜用。

### ✍️ 自我檢核

1. REST 與 gRPC 的六個維度差異？各在什麼情境選哪個？
2. Google 的資源導向設計中，標準方法有哪五個？自訂方法怎麼命名？
3. 為什麼要用 cursor 分頁而不是 offset？
4. `PATCH` 與 `update_mask` 解決什麼問題？
5. 列出三個向後相容與三個破壞相容的變更。
6. 破壞性變更的正確發布流程（四步）？
7. `POST` 怎麼做到可安全重試？

## 🔗 相關

- [[API 管理 Apigee 與 API Gateway]]
- [[Cloud API 呼叫最佳實務]]
- [[驗證與授權 ADC OAuth JWT]]
- [[韌性模式 重試 冪等 退避 斷路器]]
- [[Cloud Run]]
- [[Cloud Service Mesh 與 Network Policy]]
- [[Load Balancing 與 Session Affinity]]
- [[Cloud Deploy 與部署策略]]
- [[Section 1 設計可擴充安全可靠的雲端原生應用]]
