---
title: kubectl 與 YAML 速查
tags:
  - gcp/pcd
  - cheatsheet
  - service/gke
status: 未讀
confidence: 1
updated: 2026-09-27
---

# kubectl 與 YAML 速查

> [!tip] PCD 需要的 K8s 深度
> 不是 CKA。你需要的是：**部署一個應用、設好 probe 與資源、讓它自動擴充、診斷為什麼壞掉**。

---

## 🔍 診斷指令（**最重要，先看這段**）

```bash
kubectl get pods -o wide                      # 狀態 + 節點 + IP
kubectl describe pod POD | tail -30           # ⭐ Events 在最下面，90% 的答案在這裡
kubectl logs POD -c CONTAINER                 # 目前的 log
kubectl logs POD --previous                   # ⭐ 上一次崩潰的 log（CrashLoopBackOff 必用）
kubectl logs -l app=api --tail=50 -f          # 依標籤跟隨多個 Pod
kubectl get events --sort-by=.lastTimestamp | tail -20

kubectl get endpoints SVC                     # ⭐ 空的 = selector 錯 或 readiness 沒過
kubectl exec -it POD -- sh                    # 進去看看
kubectl port-forward POD 8080:8080            # 本機直接測 Pod
kubectl top pods && kubectl top nodes         # 實際資源用量（需 metrics-server）

kubectl get pod POD -o yaml                   # 完整定義（看實際生效的設定）
kubectl rollout status deploy/api
kubectl rollout history deploy/api
kubectl rollout undo deploy/api               # ⭐ 回滾
kubectl rollout restart deploy/api            # 重啟所有 Pod（讀新的 ConfigMap）
```

### 症狀 → 第一步
| 症狀 | 先做什麼 |
|---|---|
| `ImagePullBackOff` | `describe pod` → 檢查映像路徑與節點 SA 的 `artifactregistry.reader` |
| `CrashLoopBackOff` | `logs --previous` → 通常是啟動失敗或 liveness 太早 |
| `Pending` | `describe pod` Events → 資源不足 / taint / nodeSelector |
| `OOMKilled` | `describe pod` → 提高 `limits.memory` 或修洩漏 |
| Service 連不到 | **`get endpoints`** → selector 或 readiness |
| 502 from Ingress | 健康檢查 vs readinessProbe 路徑不一致 |
| HPA 不擴充 | `describe hpa` → 通常缺 `requests.cpu` |

---

## 📦 常用操作

```bash
kubectl apply -f deploy.yaml
kubectl apply -k ./overlays/prod              # Kustomize
kubectl diff -f deploy.yaml                   # 套用前先看差異
kubectl delete -f deploy.yaml

kubectl scale deploy/api --replicas=5
kubectl set image deploy/api app=IMG:v2
kubectl set env deploy/api LOG_LEVEL=debug
kubectl autoscale deploy/api --min=2 --max=10 --cpu-percent=70

kubectl create configmap app-cfg --from-literal=LOG_LEVEL=info --from-file=config.yaml
kubectl create secret generic db --from-literal=password=xxx    # ⚠️ 建議改用 Secret Manager CSI

kubectl config get-contexts
kubectl config use-context CTX
kubectl config set-context --current --namespace=prod
```

---

## 📄 完整的 Deployment 範本（**考試等級的正確寫法**）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels: { app: api }
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app: api }
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0            # 不減少可用 Pod（最安全）
  template:
    metadata:
      labels: { app: api, version: v2 }
    spec:
      serviceAccountName: api-ksa                    # ⭐ Workload Identity
      terminationGracePeriodSeconds: 45              # > preStop + 最長請求時間
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:                     # 分散到不同可用區
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: api } }
      containers:
        - name: app
          image: asia-east1-docker.pkg.dev/PROJECT/apps/api@sha256:abcd   # ⭐ 用 digest
          ports: [{ name: http, containerPort: 8080 }]
          resources:
            requests: { cpu: 250m, memory: 256Mi }   # 排程依據
            limits:   { cpu: "1",  memory: 512Mi }   # 超過記憶體 → OOMKilled
          env:
            - name: LOG_LEVEL
              valueFrom: { configMapKeyRef: { name: app-cfg, key: LOG_LEVEL } }
          envFrom:
            - configMapRef: { name: app-cfg }
          volumeMounts:
            - name: secrets
              mountPath: /secrets
              readOnly: true
          startupProbe:                              # 啟動慢 → 給到 5 分鐘
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            failureThreshold: 30
          readinessProbe:                            # 可以檢查下游
            httpGet: { path: /ready, port: http }
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          livenessProbe:                             # 只看自己（不查下游！）
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          lifecycle:
            preStop:
              exec: { command: ["sh", "-c", "sleep 10"] }   # 給 LB 時間移除後端
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io          # ⭐ 從 Secret Manager 取
            readOnly: true
            volumeAttributes: { secretProviderClass: app-secrets }
```

---

## 🌐 Service / Ingress / Gateway

```yaml
# ClusterIP（內部，預設）
apiVersion: v1
kind: Service
metadata:
  name: api
  annotations:
    cloud.google.com/neg: '{"ingress": true}'      # 容器原生負載平衡（Zonal NEG）
spec:
  type: ClusterIP
  selector: { app: api }
  ports: [{ name: http, port: 80, targetPort: 8080 }]
---
# Internal LB（VPC 內的 L4）
apiVersion: v1
kind: Service
metadata:
  name: api-internal
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
---
# Ingress（建立 GCP Application LB）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ing
  annotations:
    kubernetes.io/ingress.class: "gce"
    networking.gke.io/managed-certificates: "api-cert"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend: { service: { name: api, port: { number: 80 } } }
---
# Gateway API（Ingress 的後繼者，支援權重分流）
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: api-route }
spec:
  parentRefs: [{ name: external-gw }]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /v1 } }]
      backendRefs:
        - { name: api-v1, port: 80, weight: 90 }     # canary
        - { name: api-v2, port: 80, weight: 10 }
```

---

## 📈 HPA / VPA / PDB

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
    - type: External                                  # ⭐ Pub/Sub 積壓
      external:
        metric:
          name: pubsub.googleapis.com|subscription|num_undelivered_messages
          selector: { matchLabels: { resource.labels.subscription_id: orders-sub } }
        target: { type: AverageValue, averageValue: "100" }
  behavior:
    scaleDown: { stabilizationWindowSeconds: 300 }    # 防抖動
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: { name: api-vpa }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  updatePolicy: { updateMode: "Off" }                 # 先只看建議，不自動改
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api-pdb }
spec:
  minAvailable: 2                                     # ⚠️ 不要等於 replicas
  selector: { matchLabels: { app: api } }
```

---

## 🔐 Workload Identity / RBAC / NetworkPolicy

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-ksa
  annotations:
    iam.gke.io/gcp-service-account: api-gsa@PROJECT.iam.gserviceaccount.com
---
# RBAC：只能讀自己 namespace 的 Pod
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { namespace: dev, name: pod-reader }
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { namespace: dev, name: dev-readers }
subjects: [{ kind: User, name: dev@example.com, apiGroup: rbac.authorization.k8s.io }]
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
---
# 預設拒絕 ingress + 只放行 frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-ingress, namespace: prod }
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-frontend, namespace: prod }
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: frontend } } }]
      ports: [{ protocol: TCP, port: 8080 }]
```

> [!warning] egress policy 記得放行 DNS
> 加了 egress policy 後 Pod 什麼都連不到 → 忘了允許 `kube-system` 的 **UDP 53**。

---

## 🗂 其他工作負載

```yaml
# StatefulSet（有狀態，穩定名稱 + 持久磁碟）
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: kafka }
spec:
  serviceName: kafka-headless          # 需搭 Headless Service
  replicas: 3
  selector: { matchLabels: { app: kafka } }
  template:
    metadata: { labels: { app: kafka } }
    spec:
      containers:
        - name: kafka
          image: kafka:3.7
          volumeMounts: [{ name: data, mountPath: /var/lib/kafka }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 100Gi } }
---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata: { name: nightly-etl }
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Taipei"
  concurrencyPolicy: Forbid            # 不要重疊執行
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: etl-ksa
          containers:
            - name: etl
              image: IMG
```

## 🔗 相關

- [[GKE 基礎與 Autopilot]]
- [[GKE 工作負載 健康檢查與自動擴充]]
- [[Cloud Service Mesh 與 Network Policy]]
- [[Workload Identity Federation]]
- [[Load Balancing 與 Session Affinity]]
- [[gcloud 速查]]
- [[數字與限制速記]]
- [[Lab 03 GKE 部署與 HPA]]
