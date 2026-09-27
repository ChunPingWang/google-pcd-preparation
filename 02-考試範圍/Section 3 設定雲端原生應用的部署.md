---
title: Section 3 設定雲端原生應用的部署
tags:
  - gcp/pcd
  - exam/s3
weight: 24
status: 未讀
confidence: 1
updated: 2026-09-27
---

# Section 3 — 設定雲端原生應用的部署（~24%）

> [!abstract] 這一節在考什麼
> **只有兩個目標平台：Cloud Run 與 GKE。** 官方把整個部署章節收斂到這兩者，等於明示：
> 把 [[Cloud Run]] 和 [[GKE 工作負載 健康檢查與自動擴充]] 讀透，這 24% 幾乎全拿。
> 這節的題目比 Section 1 更「具體」：常出現 `gcloud` 參數、YAML 欄位、probe 設定值。

---

## 3.1 部署應用到 Cloud Run

| 官方考點 | 對應筆記 | 一句話重點 |
|---|---|---|
| **從原始碼部署**應用 | [[Cloud Run]] | `gcloud run deploy --source .` → 背後用 Cloud Build + Buildpacks，不需要 Dockerfile |
| 用**觸發器**調用 Cloud Run 服務（Eventarc、Pub/Sub） | [[Eventarc]]、[[Pub Sub]]、[[Cloud Run]] | push 訂閱 / Eventarc trigger；都需要 `roles/run.invoker` |
| 設定**事件接收端**（Eventarc、Pub/Sub） | [[Eventarc]] | CloudEvents 格式、trigger 的三種事件來源 |
| 應用中 API 的**版本化、對外開放與保護**（Apigee） | [[API 管理 Apigee 與 API Gateway]]、[[API 設計 REST 與 gRPC]] | 版本放 path（`/v1`）或 header；保護用 API key / OAuth / mTLS |

### 🎯 3.1 必背指令組
```bash
# 從原始碼部署（無 Dockerfile，Buildpacks 自動偵測）
gcloud run deploy my-svc --source . --region asia-east1

# 部署但不接流量，掛一個測試用 tag
gcloud run deploy my-svc --image IMG --no-traffic --tag canary

# 漸進式切流量：10% 給 canary
gcloud run services update-traffic my-svc --to-tags canary=10

# 全部切過去
gcloud run services update-traffic my-svc --to-latest

# 立刻回滾到指定 revision
gcloud run services update-traffic my-svc --to-revisions my-svc-00007-abc=100
```
完整版見 [[gcloud 速查]]。

### 🎯 3.1 的隱形考點
> [!important] 四個最常考的 Cloud Run 設定
> 1. **並行（`--concurrency`）**：預設 80。CPU 密集或程式非執行緒安全 → 降低；I/O 等待多 → 維持或提高（上限 1000）🔢
> 2. **最小實例（`--min-instances`）**：消除冷啟動的唯一手段，代價是持續計費
> 3. **CPU 配置**：`--cpu-throttling`（僅請求期間，便宜）vs `--no-cpu-throttling`（always allocated，背景工作/連線池才需要）
> 4. **服務身分（`--service-account`）**：決定這個服務能存取什麼；不設會用 default compute SA（**權限過大，考試裡是錯的**）

---

## 3.2 部署容器到 GKE

| 官方考點 | 對應筆記 | 一句話重點 |
|---|---|---|
| 部署容器化應用 | [[GKE 基礎與 Autopilot]]、[[GKE 工作負載 健康檢查與自動擴充]] | Deployment + Service + Ingress/Gateway 三件套 |
| 實作 **Kubernetes health check** 以提升可用性 | [[GKE 工作負載 健康檢查與自動擴充]] | `startupProbe` / `readinessProbe` / `livenessProbe` 各司其職 |
| 納入 **Horizontal Pod Autoscaler** 的屬性（擴充、指標） | [[GKE 工作負載 健康檢查與自動擴充]] | HPA 需要 `resources.requests.cpu` 才能用 CPU 百分比 |

### 🎯 3.2 probe 對照表（**必背**）
| Probe | 失敗時發生什麼 | 典型用途 | 常見錯誤 |
|---|---|---|---|
| `startupProbe` | 持續失敗 → 重啟容器；**成功前停用其他兩個 probe** | 啟動慢的應用（JVM、載入模型） | 沒設 → liveness 在啟動期就把容器殺掉 |
| `readinessProbe` | Pod 從 Service endpoints **移除**（不重啟） | 暖機、等待下游、暫時過載 | 拿 liveness 當 readiness 用 → 一有下游故障就整批重啟 |
| `livenessProbe` | **重啟容器** | 偵測死鎖、不可恢復狀態 | 檢查了下游依賴 → 造成連鎖重啟 |

```yaml
# 三兄弟的正確搭配
startupProbe:                 # 給它 5 分鐘慢慢起來
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10
readinessProbe:               # 只檢查「我現在能收流量嗎」（含下游）
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
livenessProbe:                # 只檢查「我這個 process 還活著嗎」（不查下游）
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
```

### 🎯 HPA 的三個考點
1. **指標來源**：CPU/記憶體（`resource`）、自訂指標（`pods` / `object`，透過 Custom Metrics Adapter 接 Cloud Monitoring）、外部指標（如 Pub/Sub 未處理訊息數 ← **常考**）
2. **前置條件**：用 CPU 百分比時，Pod **必須設 `requests.cpu`**，否則 HPA 算不出百分比
3. **與 Cluster Autoscaler 的關係**：HPA 加 Pod → 節點不夠 → CA 加節點。兩者是接力，不是競爭

---

## 🔁 Cloud Run vs GKE 部署對照（Day 10 自己默寫這張表）

| 維度 | Cloud Run | GKE |
|---|---|---|
| 部署單位 | revision（不可變） | Deployment 的 ReplicaSet |
| 流量切換 | 內建百分比分流 + tag | Service/Ingress 切換、兩組 Deployment、Gateway、或 Service Mesh |
| 回滾 | `update-traffic --to-revisions`（秒級） | `kubectl rollout undo` |
| 擴充 | 內建，依請求數/CPU，可縮到 0 | HPA + CA，最少通常 ≥1 |
| 健康檢查 | 可選（startup/liveness probe） | probe 是核心機制 |
| 網路 | 預設公開 URL；Direct VPC egress 進 VPC | 叢集內 DNS、Service、NetworkPolicy |
| Sidecar | 支援多容器，但無 DaemonSet | 完整支援 |
| 適合 | 無狀態 HTTP/事件服務 | 複雜拓撲、有狀態、需要 K8s 生態 |

---

## ✍️ Section 3 自我檢核

1. 只給 5% 流量到新版本、其餘留在舊版，Cloud Run 與 GKE 各怎麼做？
2. 一個 Spring Boot 服務啟動要 50 秒，部署到 GKE 後一直 CrashLoopBackOff，為什麼？怎麼修？
3. 要依「Pub/Sub 訂閱的未確認訊息數」自動擴充 GKE 工作負載 → 需要哪些元件？
4. Cloud Run 服務需要在回應送出後繼續做背景工作，該怎麼設定？
5. Eventarc 觸發 Cloud Run，需要哪些 IAM 角色給哪個身分？
6. `gcloud run deploy --source .` 與 `--image` 的差別是什麼？前者背後用到哪些服務？

## 🔗 相關

- [[Section 1 設計可擴充安全可靠的雲端原生應用]]
- [[Section 4 整合 Google Cloud 服務]]
- [[情境題 Section 2 與 3]]
- [[Lab 01 Cloud Run 端到端]]
- [[Lab 03 GKE 部署與 HPA]]
- [[Cloud Deploy 與部署策略]]
- [[00 服務索引]]
