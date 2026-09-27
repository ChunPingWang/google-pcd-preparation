---
title: Lab 03 GKE 部署與 HPA
tags:
  - gcp/pcd
  - lab
  - service/gke
status: 未做
confidence: 1
預估時間: 75 分鐘
updated: 2026-09-27
---

# Lab 03：GKE 部署、健康檢查與 HPA

> [!abstract] 你會學到
> Autopilot 叢集、三種 probe 的實際行為（**故意做出 CrashLoopBackOff 再修好**）、requests/limits 與 QoS、HPA 依 CPU 與**外部指標（Pub/Sub 積壓）**擴充、Workload Identity、PDB。
> 對應筆記：[[GKE 基礎與 Autopilot]]、[[GKE 工作負載 健康檢查與自動擴充]]

> [!warning] 成本提醒
> GKE 叢集**持續計費**。做完務必刪除叢集（見 🧹 清理）。Autopilot 相對便宜但仍會計費。

---

## 0️⃣ 建立 Autopilot 叢集

```bash
export PROJECT=$(gcloud config get-value project)
export REGION=asia-east1
gcloud services enable container.googleapis.com artifactregistry.googleapis.com

gcloud container clusters create-auto lab03 --region=$REGION      # 約 5～8 分鐘
gcloud container clusters get-credentials lab03 --region=$REGION
kubectl get nodes
```

---

## 1️⃣ 建置一個「啟動很慢」的應用

`app.py`
```python
import os, time, threading, math
from flask import Flask
app = Flask(__name__)

STARTUP_DELAY = int(os.environ.get("STARTUP_DELAY", "40"))
ready = False

def warmup():
    global ready
    time.sleep(STARTUP_DELAY)      # 模擬載入模型 / JVM 暖機
    ready = True
threading.Thread(target=warmup, daemon=True).start()

@app.get("/healthz")               # liveness: 只看 process 活著
def healthz():
    return "ok", 200

@app.get("/ready")                 # readiness: 暖機完成才收流量
def readyz():
    return ("ready", 200) if ready else ("warming up", 503)

@app.get("/")
def root():
    return "hello", 200

@app.get("/burn")                  # 燒 CPU，觸發 HPA
def burn():
    t0 = time.time()
    x = 0.0
    while time.time() - t0 < 0.5:
        x += math.sqrt(12345.6789)
    return "burned", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)), threaded=True)
```
`requirements.txt`：`Flask==3.0.3` / `gunicorn==22.0.0`

`Dockerfile`
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
ENV PORT=8080
USER 1000
CMD ["gunicorn", "-b", "0.0.0.0:8080", "-w", "1", "--threads", "8", "app:app"]
```

```bash
gcloud artifacts repositories create lab03 --repository-format=docker --location=$REGION
IMG=$REGION-docker.pkg.dev/$PROJECT/lab03/slowapp:v1
gcloud builds submit --tag $IMG .
```

---

## 2️⃣ 故意做錯：沒有 startupProbe

`deploy-bad.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: slowapp }
spec:
  replicas: 2
  selector: { matchLabels: { app: slowapp } }
  template:
    metadata: { labels: { app: slowapp } }
    spec:
      containers:
        - name: app
          image: IMAGE_PLACEHOLDER
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          livenessProbe:                    # ❌ 啟動要 40 秒，這裡 15 秒就殺
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
```

```bash
sed "s|IMAGE_PLACEHOLDER|$IMG|" deploy-bad.yaml | kubectl apply -f -
kubectl get pods -w        # 觀察 CrashLoopBackOff
kubectl describe pod -l app=slowapp | tail -25    # Events 顯示 Liveness probe failed
```
> [!danger] 學到什麼
> liveness probe 的容忍時間 = `initialDelay + failureThreshold × period` = `5 + 2×5 = 15 秒` < 40 秒啟動時間
> → 容器在還沒起來就被殺 → **CrashLoopBackOff**。這是最常見的 GKE 事故之一。

---

## 3️⃣ 修好：三種 probe 各司其職

`deploy-good.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: slowapp }
spec:
  replicas: 2
  selector: { matchLabels: { app: slowapp } }
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata: { labels: { app: slowapp } }
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          image: IMAGE_PLACEHOLDER
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          startupProbe:                     # ✅ 給 5 分鐘慢慢啟動
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:                   # ✅ 暖機完成才接流量
            httpGet: { path: /ready, port: 8080 }
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:                    # ✅ 只看 process（startup 成功後才啟用）
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop: { exec: { command: ["sh", "-c", "sleep 10"] } }
---
apiVersion: v1
kind: Service
metadata: { name: slowapp }
spec:
  type: ClusterIP
  selector: { app: slowapp }
  ports: [{ port: 80, targetPort: 8080 }]
```

```bash
sed "s|IMAGE_PLACEHOLDER|$IMG|" deploy-good.yaml | kubectl apply -f -
kubectl get pods -w                         # Running 但 READY 0/1（暖機中）
kubectl get endpoints slowapp               # ⭐ 暖機期間是空的！
sleep 45
kubectl get endpoints slowapp               # 現在有 Pod IP 了
```
> [!success] 學到什麼
> `kubectl get endpoints` 是空的 = **readiness 沒通過或 selector 不對**。這是「Service 連不到」的第一診斷。

---

## 4️⃣ HPA：依 CPU 擴充

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: slowapp-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: slowapp }
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 50 }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 120 }
EOF

# 壓測
kubectl run loadgen --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://slowapp/burn; done'
kubectl get hpa slowapp-hpa -w          # 觀察 TARGETS 與 REPLICAS 上升
```

**實驗：移除 requests 會怎樣？**
```bash
kubectl set resources deploy/slowapp --requests=cpu=0 2>/dev/null
kubectl describe hpa slowapp-hpa | grep -A3 "Metrics\|Conditions"
# → 會看到 HPA 無法計算 CPU utilization（Autopilot 可能自動補回 requests）
```
> [!important] 學到什麼
> **HPA 用 CPU 百分比時，Pod 必須有 `requests.cpu`** —— 百分比是相對於 requests 算的。
> 這是「設了 HPA 卻不擴充」最常見的原因。

```bash
kubectl delete pod loadgen --now
```

---

## 5️⃣ HPA：依 Pub/Sub 積壓擴充（**高階考點**）

```bash
gcloud services enable pubsub.googleapis.com monitoring.googleapis.com
gcloud pubsub topics create lab03-jobs
gcloud pubsub subscriptions create lab03-sub --topic=lab03-jobs

# 安裝 Custom Metrics Stackdriver Adapter
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: worker-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: slowapp }
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: pubsub.googleapis.com|subscription|num_undelivered_messages
          selector:
            matchLabels:
              resource.labels.subscription_id: lab03-sub
        target: { type: AverageValue, averageValue: "10" }
EOF

# 灌訊息造成積壓
for i in $(seq 200); do gcloud pubsub topics publish lab03-jobs --message="job-$i" >/dev/null; done
kubectl get hpa worker-hpa -w
```
> [!success] 學到什麼
> 「依佇列積壓擴充工作者」= **External metric + Custom Metrics Adapter**。
> 這是考題最愛的 HPA 進階題型。

---

## 6️⃣ Workload Identity：Pod 存取 GCP 服務

```bash
gcloud iam service-accounts create lab03-gsa
GSA=lab03-gsa@$PROJECT.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$GSA" --role=roles/pubsub.subscriber

kubectl create serviceaccount lab03-ksa
gcloud iam service-accounts add-iam-policy-binding $GSA \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:$PROJECT.svc.id.goog[default/lab03-ksa]"
kubectl annotate serviceaccount lab03-ksa iam.gke.io/gcp-service-account=$GSA

# 驗證：在 Pod 內確認身分
kubectl run wi-test --rm -it --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"lab03-ksa"}}' \
  --image=google/cloud-sdk:slim -- gcloud auth list
# 應顯示 lab03-gsa@... ✅ 完全沒有金鑰檔案
```

---

## 7️⃣ PodDisruptionBudget

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: slowapp-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: slowapp } }
EOF
kubectl get pdb
```
> 這保證節點升級/縮容時至少有 2 個 Pod 在服務。**若 `minAvailable` = 副本數，節點永遠無法排空** → 常見誤設。

---

## ✅ 檢核清單

- [ ] 重現了 CrashLoopBackOff，並能算出 liveness 的容忍時間公式
- [ ] 用 startupProbe 修好，並觀察到 `endpoints` 在 readiness 通過前是空的
- [ ] HPA 依 CPU 成功擴充
- [ ] 實證「沒有 `requests.cpu` → HPA 失效」
- [ ] 完成依 Pub/Sub 積壓的 External metric HPA
- [ ] Workload Identity 驗證成功（Pod 內 `gcloud auth list` 顯示 GSA）
- [ ] 建立 PDB 並能說明誤設的後果

## 🧹 清理（**很重要，叢集會持續計費**）

```bash
kubectl delete hpa slowapp-hpa worker-hpa --ignore-not-found
gcloud container clusters delete lab03 --region=$REGION --quiet
gcloud pubsub subscriptions delete lab03-sub --quiet
gcloud pubsub topics delete lab03-jobs --quiet
gcloud artifacts repositories delete lab03 --location=$REGION --quiet
gcloud iam service-accounts delete $GSA --quiet
```

## 🔗 相關

- [[GKE 基礎與 Autopilot]]
- [[GKE 工作負載 健康檢查與自動擴充]]
- [[Workload Identity Federation]]
- [[Pub Sub]]
- [[Cloud Monitoring 與 SLO]]
- [[Artifact Registry]]
- [[kubectl 與 YAML 速查]]
- [[Section 3 設定雲端原生應用的部署]]
- [[25 天衝刺計劃]]
