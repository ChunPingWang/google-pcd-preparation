---
title: 情境題 Section 1
tags:
  - gcp/pcd
  - practice
  - exam/s1
status: 未做
confidence: 1
題數: 20
updated: 2026-09-27
---

# 情境題：Section 1（設計可擴充、安全、可靠的雲端原生應用）

> [!abstract] 使用方式
> 1. **先遮住答案**，自己選，並寫下「我依據哪個關鍵詞」。
> 2. 對答案時**重點看「判準」**，不是看對錯。
> 3. 答錯的題目 → 回對應筆記，並在那裡加 `#weak`。
>
> 這些是依官方考試指南自製的情境題，**風格模仿真題**（長敘述 + 兩個看似都對的選項）。

---

## Q1
你要把一個容器化的無狀態 REST API 部署到 Google Cloud。流量有明顯的日夜差異，半夜幾乎沒有請求。團隊只有 3 個人，希望維運負擔最小，且希望沒有流量時不要付費。

**A.** 部署到 GKE Standard，設定 Cluster Autoscaler 最小 1 個節點
**B.** 部署到 Cloud Run，設定 `--min-instances=0`
**C.** 部署到 Compute Engine MIG，用自動擴充依 CPU 調整
**D.** 部署到 GKE Autopilot，設定 HPA `minReplicas=1`

> [!success]- 答案與判準
> **B**
> **判準**：`stateless` + `containerized` + `minimal operational overhead` + **`no charge when idle`** → 只有 Cloud Run 能縮到 0。
> A/D 的 GKE 都至少要有 1 個節點/Pod 持續計費；C 更重。
> → [[決策樹 運算平台選型]]、[[Cloud Run]]

---

## Q2
你的服務需要在每個節點上執行一個第三方安全代理程式，該代理需要存取節點的核心層級指標與特權操作。

**A.** Cloud Run 的 sidecar 容器
**B.** GKE Autopilot + DaemonSet
**C.** GKE Standard + DaemonSet
**D.** Cloud Run Jobs 定期執行

> [!success]- 答案與判準
> **C**
> **判準**：`每個節點` + `特權操作` → 需要 DaemonSet 且需要節點層級特權 → **Autopilot 限制特權容器**，必須用 Standard。
> Cloud Run 完全沒有「節點」概念。
> → [[GKE 基礎與 Autopilot]]

---

## Q3
一個全球電商平台需要關聯式資料庫，要求：跨區域的庫存扣減必須強一致、支援 SQL 交易、資料量會成長到數十 TB、需要 99.999% 可用性。

**A.** Cloud SQL for PostgreSQL（HA）+ 跨區 read replica
**B.** Spanner multi-region
**C.** Firestore 多區域
**D.** AlloyDB + 多個 read pool

> [!success]- 答案與判準
> **B**
> **判準**：`global` + `strong consistency` + `SQL transactions` + **`99.999%`** → 只有 Spanner multi-region 同時滿足。
> A 的 read replica 是**最終一致**且不能寫；C 不是關聯式；D 無法跨區強一致。
> → [[Spanner]]、[[決策樹 資料庫選型]]

---

## Q4
IoT 平台每天接收 20 億筆感測器讀數。主要查詢是「某個裝置在某個時間範圍的讀數」。需要單位數毫秒的讀取延遲，不需要 SQL 或 JOIN。

**A.** Firestore，文件 ID 用 `deviceId_timestamp`
**B.** Bigtable，row key 用 `timestamp#deviceId`
**C.** Bigtable，row key 用 `deviceId#reverseTimestamp`
**D.** BigQuery，依 timestamp 分區

> [!success]- 答案與判準
> **C**
> **判準**：`20 億筆/天` + `單位數毫秒` + 不需 SQL → **Bigtable**。
> B 的 row key **以 timestamp 開頭 → 寫入熱點**（所有新資料擠在最後一個 tablet）。
> C 把高基數的 `deviceId` 放前面（field promotion），並用 reverse timestamp 讓最新資料排前面 → 支援前綴掃描。
> D 是分析型，延遲秒級。A 撞到 Firestore 的寫入速率限制。
> → [[Bigtable]]

---

## Q5
使用者要上傳最大 2 GB 的影片。你希望不要讓上傳流量經過你的 Cloud Run 服務，同時只允許已登入使用者上傳到自己的路徑。

**A.** 讓前端把檔案 POST 到 Cloud Run，後端再寫入 Cloud Storage
**B.** 把 bucket 設為公開可寫入
**C.** 後端驗證使用者後，產生指定路徑與 method=PUT 的 V4 signed URL，前端直接上傳
**D.** 給每個使用者一個服務帳戶金鑰，讓前端直接用 SDK 上傳

> [!success]- 答案與判準
> **C**
> **判準**：`不經過後端` + `限定路徑` + `限時` → **signed URL**。
> A 讓應用承擔 2 GB 流量（且撞到請求大小上限 32 MiB）；B 是嚴重漏洞；D 絕對不可以把 SA 金鑰給客戶端。
> 補充：簽署需要 SA 有 `iam.serviceAccounts.signBlob`（`roles/iam.serviceAccountTokenCreator`）。
> → [[Cloud Storage]]

---

## Q6
你的 Cloud Run 服務需要連線到只有私有 IP 的 Memorystore Redis 實例來存放 session。

**A.** 把 Memorystore 改成公開 IP 並設定授權網路
**B.** 設定 Cloud Run 的 Direct VPC egress（或 Serverless VPC Access connector）
**C.** 在 Cloud Run 上設定 `--ingress=internal`
**D.** 用 Cloud NAT 讓 Memorystore 可以被存取

> [!success]- 答案與判準
> **B**
> **判準**：Memorystore **只有私有 IP** → serverless 必須進 VPC。
> A 不可行（Memorystore 沒有公開 IP 選項）；C 控制的是**入向**流量不是出向；D 方向相反（NAT 是讓內部連外）。
> → [[VPC 連線 Serverless VPC Access 與 Direct VPC Egress]]、[[Session 管理]]

---

## Q7
稽核法規要求交易紀錄保存 7 年，且在這 7 年內**任何人（包含專案 Owner）都不得刪除或修改**。

**A.** 開啟 Object Versioning 並設定 lifecycle 保留舊版本
**B.** 設定 bucket 的 retention policy 為 7 年並**鎖定（lock）**
**C.** 用 IAM Deny Policy 拒絕所有人的 `storage.objects.delete`
**D.** 把物件複製到另一個專案的 bucket

> [!success]- 答案與判準
> **B**
> **判準**：`任何人都不得刪除` + `固定期限` → **retention policy + lock**（**不可逆、不可縮短**）。
> A 只防誤刪，Owner 仍可刪；C 的 deny policy 可以被有權限者移除；D 不提供不可變性。
> → [[Cloud Storage]]

---

## Q8
訂單服務（Cloud Run）建立訂單後，**出貨、通知、分析**三個獨立服務都必須處理每一筆訂單事件。未來可能會再加入新的消費者，且不想修改訂單服務。

**A.** 一個 Pub/Sub topic，一個 subscription，三個 consumer 實例
**B.** 一個 Pub/Sub topic，**三個 subscription**
**C.** 三個 Cloud Tasks 佇列，訂單服務各建一個 task
**D.** 用 Workflows 依序呼叫三個服務

> [!success]- 答案與判準
> **B**
> **判準**：`每一筆都要被三個服務處理` + `未來可加消費者而不改發布者` → **扇出 = 多個 subscription**。
> A 是**負載分攤**（一則訊息只有一個 consumer 拿到）← 最常見的誘答。
> C 需要修改發布者才能加新消費者；D 是編排，耦合度高。
> → [[Pub Sub]]、[[決策樹 訊息與事件選型]]

---

## Q9
你的 Cloud Run 服務會呼叫一個第三方支付 API，該 API 限制**每秒最多 10 次呼叫**。尖峰時你的服務會收到每秒數百筆訂單。

**A.** 把訂單發到 Pub/Sub，消費者用 flow control 限制
**B.** 建立 Cloud Tasks 佇列，設定 `max-dispatches-per-second=10`，把呼叫包成 task
**C.** 在 Cloud Run 上設定 `--max-instances=10`
**D.** 用 Cloud Armor 對第三方 API 限流

> [!success]- 答案與判準
> **B**
> **判準**：`對單一目標` + **`精確的速率限制`** → **Cloud Tasks 的佇列層級速率控制**。
> A 可行但 flow control 控制的是「同時未 ack 訊息數」，不是精確 QPS；C 的實例數不等於 QPS（每個實例可併發 80）；D 保護的是你的入向流量。
> → [[Cloud Tasks]]、[[決策樹 訊息與事件選型]]

---

## Q10
使用者上傳圖片到 Cloud Storage 後，要自動產生縮圖。你希望用最少的程式碼與設定達成。

**A.** Cloud Scheduler 每分鐘掃描 bucket
**B.** Eventarc 觸發器（`google.cloud.storage.object.v1.finalized`）→ Cloud Run
**C.** 在應用裡上傳完成後同步呼叫縮圖服務
**D.** Cloud SQL trigger

> [!success]- 答案與判準
> **B**
> **判準**：`GCS 上傳後自動` → **Eventarc 直接事件**。
> A 輪詢浪費且有延遲；C 讓上傳流程耦合且失敗難重試（且若客戶端用 signed URL 直傳，後端根本不知道）；D 不存在。
> **陷阱**：縮圖**不要寫回同一個 bucket**（會無限觸發自己）。
> → [[Eventarc]]

---

## Q11
公司政策禁止建立服務帳戶金鑰。你的 GitHub Actions workflow 需要部署到 Cloud Run。

**A.** 把 SA 金鑰加密後存在 GitHub Secrets
**B.** 設定 Workload Identity Federation，並用 `attribute-condition` 限制只有 `main` 分支可用
**C.** 在 GitHub Actions 裡用 `gcloud auth login` 互動登入
**D.** 建立一個有 Editor 角色的使用者帳號給 CI 用

> [!success]- 答案與判準
> **B**
> **判準**：`外部工作負載` + `no service account keys` → **WIF**。
> **`attribute-condition` 是必要的**，否則任何 repo 都能換到你的憑證。
> A 仍是長期憑證；C 無法自動化；D 違反最小權限。
> → [[Workload Identity Federation]]

---

## Q12
你的報表儀表板每分鐘查詢一次 Spanner，查詢量很大且會影響線上交易的延遲。儀表板可以接受 10 秒前的資料。

**A.** 建立 read replica
**B.** 使用 bounded staleness 的 stale read
**C.** 把資料複製到 BigQuery
**D.** 增加 Spanner 的 processing units

> [!success]- 答案與判準
> **B**
> **判準**：`可接受稍舊的資料` + `降低對主資料庫的影響` → **stale read**（可從最近的複本讀，不佔用讀寫路徑的資源）。
> A Spanner 沒有獨立的「read replica」概念（複本是內建的）；C 是長期方案但延遲更高、複雜；D 花錢但沒解決根因。
> → [[Spanner]]、[[一致性 交易與資料複寫]]

---

## Q13
你要公開一個 API 給外部合作夥伴，需要：依方案分層的配額（免費版 1000 次/日、付費版 100 萬次/月）、開發者自助註冊與文件門戶、使用量報表。

**A.** Cloud Run + 自己實作 API key 驗證與計數
**B.** API Gateway + OpenAPI spec
**C.** Apigee
**D.** Global ALB + Cloud Armor 速率限制

> [!success]- 答案與判準
> **C**
> **判準**：`developer portal` + `API products / 分層配額` + `使用量報表` → **Apigee**（完整 API 管理平台）。
> B 的 API Gateway 是輕量閘道，沒有開發者門戶與 API 產品化；D 只能做粗粒度速率限制。
> **反向題**：若題目只說「簡單保護一個 Cloud Run API 並限流」→ 答案是 **API Gateway**（Apigee 變成誘答）。
> → [[API 管理 Apigee 與 API Gateway]]

---

## Q14
Cloud Run 服務在回應使用者後，還需要寫入稽核紀錄與呼叫兩個下游服務。你發現這些背景工作有時沒有完成。

**A.** 提高 `--timeout`
**B.** 把背景工作改用 Cloud Tasks 建立任務（或啟用 CPU always allocated）
**C.** 提高 `--concurrency`
**D.** 增加 `--max-instances`

> [!success]- 答案與判準
> **B**
> **判準**：回應送出後 CPU 被節流（預設 `cpu-throttling`）→ 背景執行緒被凍結。
> 兩個正解：① **把工作丟給 Cloud Tasks**（更 Google、有重試與可觀測性，**首選**）；② `--no-cpu-throttling`（保留背景 CPU，但沒有重試保證）。
> A/C/D 都與此無關。
> → [[Cloud Run]]、[[Cloud Tasks]]

---

## Q15
你的微服務彼此透過 HTTP 呼叫。安全團隊要求：所有服務間流量必須雙向 TLS 加密、且只有特定服務能呼叫 `POST /admin/*`。你不想修改應用程式碼。

**A.** Kubernetes NetworkPolicy
**B.** Cloud Service Mesh：`PeerAuthentication` STRICT + `AuthorizationPolicy`
**C.** VPC 防火牆規則
**D.** 在每個服務裡實作 mTLS

> [!success]- 答案與判準
> **B**
> **判準**：`mTLS` + `不改程式` + **`依 HTTP 方法與路徑授權`** → **Service Mesh**（L7 + 身分）。
> A/C 只到 L3/L4，看不到 HTTP 路徑；D 要改程式。
> **導入注意**：先用 `PERMISSIVE` 過渡再切 `STRICT`。
> → [[Cloud Service Mesh 與 Network Policy]]

---

## Q16
你的行動應用有 500 萬名使用者，需要註冊/登入（含 Google、Apple 登入與 MFA），並且使用者只能讀寫自己的資料。資料存在 Firestore，App 直接連線。

**A.** 為每個使用者建立一個 IAM 使用者與服務帳戶
**B.** Identity Platform 驗證 + Firestore Security Rules
**C.** IAP 保護 Firestore
**D.** 後端自建 session 表 + API key

> [!success]- 答案與判準
> **B**
> **判準**：`數百萬終端使用者` → **Identity Platform**（不是 IAM）；`客戶端直連且只能看自己的資料` → **Security Rules**。
> A 完全不可行（IAM 不是為終端使用者設計的）；C IAP 保護的是應用端點且用 IAM 身分；D 沒回答授權問題。
> → [[IAP Identity Platform 與 Web Security Scanner]]、[[Firestore]]

---

## Q17
你的服務每秒讀取同一份「商品分類表」數千次，該表每天只更新一次。目前每次都查 Cloud SQL，導致資料庫 CPU 很高。

**A.** 增加 Cloud SQL 的機器規格
**B.** 建立 read replica
**C.** 用 Memorystore 做 cache-aside，TTL 設 1 小時並加隨機抖動
**D.** 把表搬到 Firestore

> [!success]- 答案與判準
> **C**
> **判準**：`讀多寫少` + `降低資料庫負載` → **快取（cache-aside）**。
> **TTL 要加抖動**避免大量 key 同時過期造成雪崩。
> A 花錢沒解決根因；B 也要錢且仍有查詢成本；D 只是換一個資料庫，問題一樣。
> → [[Memorystore 與快取策略]]

---

## Q18
你的 Cloud Run 服務在高流量時出現 `OOMKilled`（記憶體不足被終止）。每個請求會載入一個約 200 MB 的資料集進行處理。服務設定：1 vCPU、2 GiB 記憶體、concurrency 80。

**A.** 提高 `--max-instances`
**B.** 降低 `--concurrency` 並/或提高 `--memory`
**C.** 提高 `--cpu`
**D.** 縮短 `--timeout`

> [!success]- 答案與判準
> **B**
> **判準**：`80 併發 × 200 MB = 16 GB` 遠超 2 GiB → 記憶體被多個請求瓜分。
> 降低 concurrency（例如 8）或提高記憶體（或兩者），並考慮讓資料集在全域載入一次而非每個請求載入。
> A/C/D 都不影響單一實例的記憶體壓力。
> → [[Cloud Run]]、[[成本與資源最佳化]]

---

## Q19
你要在單一區域內設計一個能「自動撐過整個可用區（zone）故障」的架構：Cloud Run 前端 + PostgreSQL 資料庫 + Redis 快取。

**A.** Cloud Run（自動跨 zone）+ Cloud SQL `--availability-type=REGIONAL` + Memorystore Standard tier
**B.** Cloud Run + Cloud SQL 單機 + Memorystore Basic tier
**C.** Cloud Run + Cloud SQL + 跨區 read replica + Memorystore Basic
**D.** GKE zonal cluster + Cloud SQL HA + Memorystore Standard

> [!success]- 答案與判準
> **A**
> **判準**：`zone 故障自動撐過` →
> - Cloud Run **本身就跨 zone**（不用做事）
> - Cloud SQL 要 **REGIONAL（HA）**：同 region 跨 zone 同步複寫 + 自動 failover
> - Memorystore 要 **Standard tier**（Basic 沒有複本、沒有自動 failover）
> B 的 Basic 與單機都是單點；C 的跨區 replica 需**手動 promote**（且是為 region 故障設計）；D 的 zonal cluster 本身就是單點。
> → [[一致性 交易與資料複寫]]、[[地理分布 區域與可用區設計]]

---

## Q20
你要把一個跑在 VM 上的 Java 單體應用現代化。限制：不能一次重寫、要能逐步把功能搬出來、遷移期間使用者不能感覺到差異。

**A.** 直接重寫成微服務後一次切換
**B.** 容器化後整包上 GKE，然後用 Strangler Fig 模式在前面放 LB/API Gateway，逐步把路徑導向新服務
**C.** 把單體拆成三個 Cloud Run 服務，各自複製一份程式碼
**D.** 把單體搬到 App Engine Flexible

> [!success]- 答案與判準
> **B**
> **判準**：`不能一次重寫` + `逐步` + `使用者無感` → **Strangler Fig**（前置路由層 + 漸進替換）。
> A 是 big-bang 重寫（高風險）；C 是複製而非拆分；D 只是換平台，沒有解決現代化。
> → [[Compute Engine 與 App Engine]]、[[決策樹 運算平台選型]]、[[12-Factor 與雲端原生設計原則]]

---

## 📊 作答紀錄

| 日期 | 答對 / 20 | 錯題號 | 主要錯因 |
|---|---|---|---|
| | | | |
| | | | |

**錯因分類**：判準 / 知識 / 數字 / 讀題 / 粗心 → 見 [[每日追蹤模板]]

## 🔗 相關

- [[Section 1 設計可擴充安全可靠的雲端原生應用]]
- [[情境題 Section 2 與 3]]
- [[情境題 Section 4]]
- [[常見陷阱與誘答選項識別]]
- [[決策樹 運算平台選型]]
- [[決策樹 資料庫選型]]
- [[決策樹 訊息與事件選型]]
