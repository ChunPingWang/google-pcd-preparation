---
title: 開發環境 Cloud Code Shell Workstations 與 AI 工具
tags:
  - gcp/pcd
  - service/cloud-code
  - service/cloud-workstations
  - exam/s2
  - ai
status: 未讀
confidence: 1
importance: 4
updated: 2026-09-27
---

# 開發環境：Cloud Code / Shell / Workstations 與 AI 工具

> [!abstract] 一句話定位
> 官方 Section 2.1 三條考點全在這裡，而且 **2026 版新增了大量 AI 內容**：
> - 「用 gcloud CLI **模擬** Google Cloud 服務做本地開發與單元測試」
> - 「使用 Console、Cloud SDK、**Cloud Code**、**Gemini Cloud Assist**、**Cloud Shell**、**Cloud Workstations**」
> - 「設定 IDE 的適當整合（Cloud SDK、AI 工具：coding assistant、**MCP server**）」
>
> 這是最容易被舊考古題漏掉、但最好拿分的一節。

---

## 🧪 本地模擬器（Emulators）— **最高頻考點**

```bash
# Firestore
gcloud emulators firestore start --host-port=localhost:8080
export FIRESTORE_EMULATOR_HOST=localhost:8080

# Pub/Sub
gcloud emulators pubsub start --host-port=localhost:8085
export PUBSUB_EMULATOR_HOST=localhost:8085
# 或用 env-init 自動設定
$(gcloud beta emulators pubsub env-init)

# Bigtable
gcloud emulators bigtable start
export BIGTABLE_EMULATOR_HOST=localhost:8086

# Spanner
gcloud emulators spanner start
export SPANNER_EMULATOR_HOST=localhost:9010

# Datastore（Firestore in Datastore mode）
gcloud emulators datastore start
```

| 服務 | 有官方 emulator？ |
|---|---|
| **Firestore** | ✅ |
| **Datastore** | ✅ |
| **Pub/Sub** | ✅ |
| **Bigtable** | ✅ |
| **Spanner** | ✅ |
| Cloud Tasks | ⚠️ 有（部分/beta，實務常用本機 fake） |
| **Cloud Storage** | ❌ 無官方（用第三方 fake-gcs-server 或測試 bucket） |
| **BigQuery** | ❌ 無官方（用測試 dataset） |
| Cloud SQL | ❌ 無（**直接跑本機 MySQL/PostgreSQL 容器**即可，因為它就是標準資料庫） |
| Cloud Run / GKE | 用 **minikube / Cloud Code / `docker run`** 在本機跑容器 |

> [!important] 考點記憶法
> **「有專有 API 的 NoSQL/訊息服務 → 有 emulator；有標準替代品的（SQL、物件儲存）→ 用真的或第三方。」**
> 題目問「如何在無網路/零成本下做整合測試」→ **emulator**；問 Cloud Storage → 誠實回答沒有官方 emulator，用測試 bucket。

**用戶端程式庫的行為**：設好 `*_EMULATOR_HOST` 環境變數後，**用戶端程式庫會自動改連模擬器，且不需要憑證** → 程式碼完全不用改。這點很常考。

---

## 🖥 四種開發環境

| | **Cloud Shell** | **Cloud Workstations** | **Cloud Code** | 本機 + gcloud |
|---|---|---|---|---|
| 是什麼 | 瀏覽器內的臨時 VM + 預裝工具 | **代管的、可自訂的開發 VM**，在**你的 VPC 內** | **IDE 外掛**（VS Code / JetBrains / Cloud Shell Editor） | 自己的機器 |
| 持久性 | `$HOME` 有 5 GB 持久空間；閒置會回收 | 持久、可設定機器規格與映像 | 依附於你的 IDE | 完全自己管 |
| 預裝 | `gcloud`、`kubectl`、`docker`、`terraform`、多語言 SDK | 你自訂的映像（團隊統一） | — | 自己裝 |
| 適合 | **快速一次性操作**、上課、考試練習 | **團隊統一環境**、需要存取私有資源、合規（程式碼不落地） | 在 IDE 內部署/除錯 Cloud Run 與 GKE | 日常開發 |
| 成本 | 免費（有配額） | 依 VM 計費 | 免費 | — |

### Cloud Code 能做什麼（Section 2.1 直接考）
- 一鍵部署到 **Cloud Run / GKE**，並在 IDE 內看 log。
- **本機開發迴圈**：改程式碼 → 自動重建 → 自動部署到 minikube/遠端叢集（底層用 **Skaffold**）。
- **遠端除錯**：在 Cloud Run / GKE 上的容器設中斷點。
- YAML 驗證與自動完成（Kubernetes、`cloudbuild.yaml`、`skaffold.yaml`）。
- 內建 **Gemini Code Assist**。

### Cloud Workstations 的三個賣點（考題關鍵詞）
1. **安全**：程式碼不落在個人筆電；可在 VPC 內、受 VPC-SC 保護、強制 IAM。
2. **一致**：團隊用同一個容器映像，「在我電腦上可以跑」的問題消失。
3. **可及性**：可存取**私有 GKE / 私有資料庫**，不需要 VPN。

> 考題：`regulated industry`、`source code must not be stored on developer laptops`、`consistent environment`、`access private resources` → **Cloud Workstations**。

---

## 🤖 AI 輔助開發（2026 新增考點）

| 工具 | 定位 | 能做什麼 |
|---|---|---|
| **Gemini Code Assist** | IDE 內的 AI coding assistant | 程式碼補全、產生函式、**產生單元測試**（官方明文考點）、解釋程式碼、重構建議、程式碼轉換 |
| **Gemini Cloud Assist** | **雲端維運**的 AI 助理 | 解釋錯誤與 log、診斷問題、產生查詢、建議設定、成本/架構建議 |
| **Gemini in BigQuery / Databases** | 資料面 | 產生 SQL、解釋查詢計畫、最佳化建議 |
| **MCP server**（Model Context Protocol） | 讓 AI 助理**安全地取得外部上下文與工具** | 把你的 API/資料庫/文件接給 AI 助理使用 |
| **Context engineering** | 提供正確上下文的方法 | 給模型專案規範、既有程式碼樣式、API 契約，讓產出可用 |

### 官方明文考點：「**借助 AI coding assistant 撰寫單元測試**」
實務要點（也是考試想要的觀念）：
1. 讓助理讀懂**既有測試風格**再產生新測試（context engineering）。
2. 重點放在**邊界條件與錯誤路徑** — 這是人最容易漏的。
3. **產出必須人工審查**：AI 可能產生「看起來通過但沒真的驗證」的測試（例如 assert 太寬鬆）。
4. 測試覆蓋率不等於測試品質。

### 「AI 輔助的可觀測性」（Section 4.3 考點）
用 **Gemini Cloud Assist** 在 Logging / Monitoring / Error Reporting 裡：
- 摘要一段錯誤 log 的可能根因
- 由自然語言產生 log 查詢或 MQL/PromQL
- 對 incident 提出調查步驟
> 定位：**加速人的診斷，不是取代 SLO 與監控設計**。見 [[Cloud Monitoring 與 SLO]]。

---

## 🧰 gcloud CLI 的開發者必備設定

```bash
# 多環境切換：用 configuration
gcloud config configurations create dev
gcloud config set project my-dev-project
gcloud config set run/region asia-east1
gcloud config configurations activate dev
gcloud config configurations list

# 本機取得 ADC（見 驗證與授權 筆記）
gcloud auth application-default login
gcloud auth application-default login --impersonate-service-account=api-sa@PROJECT.iam.gserviceaccount.com

# 安裝元件
gcloud components install beta kubectl skaffold
gcloud components update

# 好用的輸出格式
gcloud run services list --format="table(metadata.name, status.url)"
gcloud run services describe api --format="value(status.url)"
gcloud projects list --format=json | jq '.[].projectId'
```

---

## 🎯 考點速記

| 看到題目說… | 就想到 |
|---|---|
| `local development and unit tests without cloud costs` | **Emulator**（`gcloud emulators ...`） |
| `test Firestore + Pub/Sub logic offline` | 兩者都有 emulator，設 `*_EMULATOR_HOST` |
| `Cloud Storage 的本地測試` | **沒有官方 emulator** → 測試 bucket / 第三方 fake |
| `quick one-off gcloud command in the browser` | **Cloud Shell** |
| `consistent, secure dev environment for the team` | **Cloud Workstations** |
| `code must not be stored on laptops` | Cloud Workstations |
| `deploy & debug Cloud Run from inside my IDE` | **Cloud Code** |
| `iterate quickly on a GKE app locally` | Cloud Code + **Skaffold**（minikube） |
| `AI helps write unit tests` | **Gemini Code Assist** |
| `AI helps diagnose production issues / generate queries` | **Gemini Cloud Assist** |
| `give the AI assistant access to our internal tools` | **MCP server** |
| `switch between dev/prod projects easily` | `gcloud config configurations` |

## 💣 真實場景陷阱

1. **忘記 unset emulator 環境變數**：在本機「測試通過」其實全是打模擬器；或反過來，正式程式意外連到模擬器。
2. **以為 emulator 行為 100% 等於雲端**：模擬器不驗證 IAM、索引行為/配額/延遲都不同 → **關鍵路徑仍要在真實環境測**。
3. **Cloud Shell 當開發機**：閒置會回收、規格固定，只有 `$HOME` 持久。
4. **Security Rules 只在雲端測**：Firestore 模擬器可以測 rules（更快更安全）。
5. **AI 產生的測試沒人審**：assert 空泛、mock 過度，覆蓋率高但無效。
6. **AI 產生的程式碼含過時 API**：要對照官方文件驗證（尤其 GCP 的服務名稱與參數常改）。

## ✍️ 自我檢核

1. 哪些 Google Cloud 服務有官方 emulator？哪兩個重要服務沒有？沒有的怎麼測？
2. 設定 emulator 後，程式碼需要修改嗎？為什麼？
3. Cloud Shell、Cloud Workstations、Cloud Code 各解決什麼問題？
4. 「程式碼不能存在開發者筆電上」且「要能連私有 GKE」→ 選什麼？
5. 用 AI 產生單元測試時，有哪三個必須人工把關的點？
6. Gemini Code Assist 與 Gemini Cloud Assist 的差別？
7. Emulator 測試通過但上雲失敗，最可能的三個原因？

## 🔗 相關

- [[Cloud Build]]
- [[Firestore]]
- [[Pub Sub]]
- [[Bigtable]]
- [[Spanner]]
- [[Vertex AI Gemini API 給開發者]]
- [[Cloud Monitoring 與 SLO]]
- [[驗證與授權 ADC OAuth JWT]]
- [[gcloud 速查]]
- [[Section 2 建置與測試應用]]
