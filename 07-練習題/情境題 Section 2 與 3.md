---
title: 情境題 Section 2 與 3
tags:
  - gcp/pcd
  - practice
  - exam/s2
  - exam/s3
status: 未做
confidence: 1
題數: 18
updated: 2026-09-27
---

# 情境題：Section 2（建置與測試）+ Section 3（部署）

> 使用方式同 [[情境題 Section 1]]：先遮答案、寫下判準關鍵詞、錯題加 `#weak`。

---

## 🧪 Section 2：建置與測試

### Q1
你要在 CI 裡測試一段「寫入 Firestore 後發布 Pub/Sub 訊息」的程式碼，要求**不連線雲端、不產生費用**。

**A.** 在 Cloud Build 裡建立一個測試專案並使用真實服務
**B.** 在 Cloud Build 的 step 中啟動 **Firestore 與 Pub/Sub emulator**，設定 `FIRESTORE_EMULATOR_HOST` 與 `PUBSUB_EMULATOR_HOST`
**C.** 用 mock 取代所有 Google Cloud 用戶端
**D.** 跳過整合測試，只做單元測試

> [!success]- 答案與判準
> **B**
> **判準**：`不連雲端` + `整合測試` → **emulator**。設好環境變數後**程式碼完全不用改**，用戶端程式庫會自動改連模擬器且不需憑證。
> C 是單元測試的做法（測不到真實的查詢/索引行為）；D 放棄了覆蓋。
> → [[開發環境 Cloud Code Shell Workstations 與 AI 工具]]

---

### Q2
下列哪個服務**沒有**官方的本地模擬器？（選兩個）

**A.** Firestore　**B.** Pub/Sub　**C.** Cloud Storage　**D.** Bigtable　**E.** BigQuery　**F.** Spanner

> [!success]- 答案與判準
> **C 與 E**
> 有官方 emulator：**Firestore、Datastore、Pub/Sub、Bigtable、Spanner**。
> **Cloud Storage 與 BigQuery 沒有官方 emulator** → 用測試 bucket / 測試 dataset，或第三方 fake。
> 記憶法：**有專有 API 的 NoSQL/訊息 → 有 emulator；有標準替代品的 → 沒有。**
> → [[開發環境 Cloud Code Shell Workstations 與 AI 工具]]

---

### Q3
你的 `cloudbuild.yaml` 有 lint、unit-test、build 三個步驟。目前總時間 8 分鐘，你想縮短。lint 與 unit-test 互不相依。

**A.** 把三個步驟合併成一個
**B.** 對 lint 與 unit-test 都設 `waitFor: ['-']`，build 設 `waitFor: ['lint','unit-test']`
**C.** 提高 `timeout`
**D.** 把 build 移到另一個 trigger

> [!success]- 答案與判準
> **B**
> **判準**：Cloud Build 的步驟**預設循序執行**；`waitFor: ['-']` = 不等任何步驟 → **立即開始（平行）**。
> 其他加速手段：`--cache-from` / kaniko cache、`options.machineType: E2_HIGHCPU_8`、精簡 Dockerfile。
> → [[Cloud Build]]

---

### Q4
Cloud Build 部署 Cloud Run 時失敗，錯誤訊息包含 `iam.serviceaccounts.actAs` denied。

**A.** 給 Cloud Build 的 SA `roles/run.admin`
**B.** 給 Cloud Build 的 SA 對**執行時 SA** 的 `roles/iam.serviceAccountUser`
**C.** 給執行時 SA `roles/cloudbuild.builds.editor`
**D.** 為 Cloud Build 建立新的 SA 金鑰

> [!success]- 答案與判準
> **B**
> **判準**：`actAs` = 「以某個 SA 的身分建立資源」→ 需要對**目標 SA** 的 `roles/iam.serviceAccountUser`。
> 這是實務與考試的超高頻點。A 給的是部署權限（不含 actAs）；C 方向錯；D 完全不相關。
> → [[Cloud Build]]、[[IAM 與服務帳戶]]

---

### Q5
你要在 Cloud Build 裡使用一個 Slack webhook URL。

**A.** 放在 `substitutions` 裡
**B.** 用 `availableSecrets` 從 Secret Manager 取，並用 `secretEnv` 注入步驟
**C.** 硬寫在 `cloudbuild.yaml`
**D.** 放在建置觸發器的環境變數裡

> [!success]- 答案與判準
> **B**
> **判準**：祕密要走 **Secret Manager + `availableSecrets`/`secretEnv`**。
> A/C/D 都會讓值出現在建置設定與紀錄裡。另外 Cloud Build 的 SA 需要該祕密的 `secretAccessor`。
> → [[Cloud Build]]、[[Secret Manager 與 Cloud KMS]]

---

### Q6
安全團隊要求：只有經過 CI 建置、通過漏洞掃描的映像才能部署到生產。導入時不能影響現有部署。

**A.** 直接把 Binary Authorization 政策設為 `REQUIRE_ATTESTATION` + `ENFORCED_BLOCK_AND_AUDIT_LOG`
**B.** 先用 `DRYRUN_AUDIT_LOG_ONLY` 觀察違規，確認後再切 `ENFORCED`
**C.** 在 Artifact Registry 設定 cleanup policy
**D.** 在 Cloud Build 最後加一個 `echo "scanned"` 步驟

> [!success]- 答案與判準
> **B**
> **判準**：`導入時不能影響現有部署` → **先 dry-run**。
> A 會立刻擋掉所有沒有 attestation 的部署（包含系統元件，若沒設 allowlist/`globalPolicyEvaluationMode`）。
> 補充：Binary Auth **只能用 digest 部署**，tag 不行。
> → [[供應鏈安全 Artifact Analysis 與 Binary Authorization]]

---

### Q7
公司在受監管產業，規定「原始碼不得儲存在開發者的個人筆電」，同時開發者需要存取只有私有 IP 的 GKE 叢集與資料庫。

**A.** Cloud Shell
**B.** Cloud Workstations
**C.** Cloud Code 外掛
**D.** 每個開發者一台 Compute Engine VM 自行管理

> [!success]- 答案與判準
> **B**
> **判準**：`程式碼不落地` + `存取私有資源` + `團隊一致環境` → **Cloud Workstations**（在你的 VPC 內、可自訂映像、受 IAM/VPC-SC 控制）。
> A 是臨時環境（只有 `$HOME` 持久、規格固定、閒置回收）；C 是 IDE 外掛（本機仍有程式碼）；D 維運負擔高。
> → [[開發環境 Cloud Code Shell Workstations 與 AI 工具]]

---

### Q8
你的映像掃描出 30 個 CRITICAL 等級的 OS 套件漏洞。最有效的長期解法是？

**A.** 手動 `apt upgrade` 每個有漏洞的套件
**B.** 改用 distroless / minimal 基底映像，並在 CI pipeline 加入掃描 gate
**C.** 把漏洞標記為已接受風險
**D.** 只在生產環境關閉掃描

> [!success]- 答案與判準
> **B**
> **判準**：`最有效的長期解法` → **減少攻擊面（distroless 沒有 shell、沒有套件管理器）+ 自動化把關**。
> A 是一次性的（下週又會有新 CVE）；C/D 不是解法。
> 搭配：固定基底映像 digest、多階段建置、非 root 執行。
> → [[供應鏈安全 Artifact Analysis 與 Binary Authorization]]

---

## 🚀 Section 3：部署

### Q9
你要在 Cloud Run 上驗證新版本，但**不能讓任何真實使用者碰到它**。

**A.** 部署到另一個 Cloud Run 服務
**B.** `gcloud run deploy --no-traffic --tag candidate`，然後用 tag 專屬 URL 測試
**C.** `update-traffic --to-tags candidate=1`
**D.** 部署後立刻 rollback

> [!success]- 答案與判準
> **B**
> **判準**：`在生產環境驗證但不影響使用者` → **`--no-traffic` + `--tag`**，tag 會給你一個專屬 URL（`https://candidate---svc-xxx.a.run.app`）。
> A 環境不同（設定、SA、連線可能不一樣）；C 已經有 1% 真實使用者了。
> → [[Cloud Run]]、[[Cloud Deploy 與部署策略]]

---

### Q10
生產環境的新版本上線後錯誤率飆升，你需要**立即**回到上一版。

**A.** 重新建置舊的 commit 並部署
**B.** `gcloud run services update-traffic SERVICE --to-revisions OLD_REVISION=100`
**C.** 刪除新的 revision
**D.** `kubectl rollout undo`

> [!success]- 答案與判準
> **B**
> **判準**：revision **不可變** → 回滾只是**改流量指向**，秒級完成，不需重新建置。
> A 要好幾分鐘；C 不必要（且流量不會自動轉移）；D 是 GKE 的指令。
> → [[Cloud Run]]

---

### Q11
一個 Spring Boot 服務在 GKE 上不斷 `CrashLoopBackOff`。啟動需要約 50 秒。目前設定：`livenessProbe` 的 `initialDelaySeconds: 10`、`periodSeconds: 5`、`failureThreshold: 3`。

**A.** 提高 `replicas`
**B.** 加上 `startupProbe`（`periodSeconds: 10`、`failureThreshold: 30`）
**C.** 移除 livenessProbe
**D.** 提高 `limits.memory`

> [!success]- 答案與判準
> **B**
> **判準**：目前的容忍時間 = `10 + 3×5 = 25 秒` < 50 秒啟動 → 還沒起來就被殺。
> `startupProbe` 成功前**會停用 liveness 與 readiness**，是專門解決啟動慢的機制（容忍 `30×10 = 300 秒`）。
> C 會失去死鎖偵測能力（不是好答案）；A/D 與此無關。
> → [[GKE 工作負載 健康檢查與自動擴充]]

---

### Q12
你在 GKE 上設定了 HPA（CPU 目標 70%），但 Pod 數量完全不變。`kubectl describe hpa` 顯示 CPU 指標為 `<unknown>`。

**A.** 提高 `maxReplicas`
**B.** 為容器設定 `resources.requests.cpu`
**C.** 安裝 metrics-server（Autopilot 已內建）
**D.** 改用 VPA

> [!success]- 答案與判準
> **B**
> **判準**：CPU **Utilization 是相對於 `requests` 計算的** → 沒有 `requests.cpu` 就無法算百分比。
> 這是 HPA 不生效的**第一嫌疑**。
> → [[GKE 工作負載 健康檢查與自動擴充]]

---

### Q13
你要讓 GKE 的工作者依 **Pub/Sub 訂閱的未處理訊息數** 自動擴充。

**A.** HPA 用 `Resource` 類型的 CPU 指標
**B.** HPA 用 `External` 類型的 `pubsub.googleapis.com|subscription|num_undelivered_messages`，並安裝 Custom Metrics Stackdriver Adapter
**C.** Cluster Autoscaler
**D.** VPA

> [!success]- 答案與判準
> **B**
> **判準**：`叢集外部的指標` → **External metric** + **Custom Metrics Adapter**（把 Cloud Monitoring 指標暴露給 HPA）。
> A 只反映 CPU（訊息積壓時 CPU 可能不高）；C 是節點層級（HPA 加 Pod 後才由 CA 加節點）；D 是垂直調整。
> → [[GKE 工作負載 健康檢查與自動擴充]]、[[Pub Sub]]

---

### Q14
滾動更新期間有大量 502 錯誤。Pod 有正確的 readinessProbe。

**A.** 提高 `replicas`
**B.** 加上 `preStop` hook（`sleep 10`）並確保應用處理 `SIGTERM`，同時把 `terminationGracePeriodSeconds` 設得比 preStop + 最長請求時間更長
**C.** 把 `maxUnavailable` 設為 50%
**D.** 移除 readinessProbe

> [!success]- 答案與判準
> **B**
> **判準**：Pod 被刪除時，**LB 移除後端需要時間**；若容器立刻關閉，那段時間的請求會 502。
> 正解是 **優雅關閉**：`preStop` 給 LB 時間收斂 + 應用處理 SIGTERM 排空進行中的請求 + 足夠的 grace period。
> C 會讓情況更糟；D 錯誤方向。
> → [[GKE 工作負載 健康檢查與自動擴充]]、[[韌性模式 重試 冪等 退避 斷路器]]

---

### Q15
`kubectl get endpoints my-svc` 回傳空的，但 Pod 都是 `Running`。

**A.** Service 的 `selector` 與 Pod 的 labels 不匹配，或 readinessProbe 尚未通過
**B.** Service 的 `type` 應該改成 `LoadBalancer`
**C.** 需要建立 Ingress
**D.** 節點數不足

> [!success]- 答案與判準
> **A**
> **判準**：endpoints 空 = **沒有「符合 selector 且 Ready」的 Pod**。這兩個是唯一原因。
> 這是「Service 連不到」的**第一診斷步驟**。
> → [[kubectl 與 YAML 速查]]、[[GKE 工作負載 健康檢查與自動擴充]]

---

### Q16
你要用 Eventarc 觸發一個設定為需要驗證（`--no-allow-unauthenticated`）的 Cloud Run 服務。需要哪些 IAM 授權？（選最完整的）

**A.** 只需要觸發器的 SA 有 `roles/eventarc.admin`
**B.** 觸發器的 SA 需要目標服務的 `roles/run.invoker` 與 `roles/eventarc.eventReceiver`；若來源是 GCS，GCS 服務代理需要 `roles/pubsub.publisher`
**C.** Cloud Run 服務需要設為 `--allow-unauthenticated`
**D.** 只需要 `roles/pubsub.subscriber`

> [!success]- 答案與判準
> **B**
> **判準**：Eventarc 用**指定的 SA** 去呼叫目標 → 該 SA 需要 `run.invoker`；接收事件需要 `eventarc.eventReceiver`；GCS 事件經由 Pub/Sub → GCS 服務代理需要 publisher。
> 建立者還需要對該 SA 的 `actAs`。
> → [[Eventarc]]

---

### Q17
你要在 Cloud Run 上部署一個 API，並確保**只有 API Gateway 能呼叫它**。

**A.** 只在 API Gateway 設定 API key
**B.** Cloud Run 設 `--no-allow-unauthenticated`，並在 OpenAPI 的 `x-google-backend` 設 `jwt_audience`，讓 Gateway 用自己的 SA 取得 ID token
**C.** 用 Cloud Armor 封鎖所有 IP 除了 Gateway
**D.** Cloud Run 設 `--ingress=all`

> [!success]- 答案與判準
> **B**
> **判準**：**後端必須要求驗證**，Gateway 用自己的服務身分取得 ID token 呼叫 → 其他人直接打後端會 403。
> A 只保護 Gateway 入口，後端仍公開（可被繞過）；C Gateway 的出口 IP 不固定；D 相反。
> → [[API 管理 Apigee 與 API Gateway]]、[[Cloud Run]]

---

### Q18
你需要把**同一個映像**依序部署到 dev → staging → prod，prod 前需要人工核准，且 prod 要用 25% → 50% → 100% 的漸進式發布並能一鍵回滾。

**A.** 三個 Cloud Build 觸發器各自建置並部署
**B.** Cloud Deploy delivery pipeline：三個 target、prod 設 `requireApproval: true`、canary strategy `percentages: [25, 50]`
**C.** 三個 Cloud Run 服務手動部署
**D.** 用 Workflows 編排部署

> [!success]- 答案與判準
> **B**
> **判準**：`同一個產出物晉升過多環境` + `核准` + `代管 canary` + `回滾` → **Cloud Deploy**。
> A 會**重新建置**（不同的產出物，違反「build once, deploy many」）；C 無法審計與自動化；D 可行但要自己實作全部機制。
> 記住分工：**Cloud Build = CI（產生產出物）、Cloud Deploy = CD（晉升產出物）**。
> → [[Cloud Deploy 與部署策略]]

---

## 📊 作答紀錄

| 日期 | 答對 / 18 | 錯題號 | 主要錯因 |
|---|---|---|---|
| | | | |

## 🔗 相關

- [[Section 2 建置與測試應用]]
- [[Section 3 設定雲端原生應用的部署]]
- [[情境題 Section 1]]
- [[情境題 Section 4]]
- [[常見陷阱與誘答選項識別]]
