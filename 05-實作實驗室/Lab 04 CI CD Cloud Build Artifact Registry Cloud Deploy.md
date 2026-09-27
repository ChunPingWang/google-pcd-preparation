---
title: Lab 04 CI CD Cloud Build Artifact Registry Cloud Deploy
tags:
  - gcp/pcd
  - lab
  - service/cloud-build
  - service/cloud-deploy
status: 未做
confidence: 1
預估時間: 75 分鐘
updated: 2026-09-27
---

# Lab 04：CI/CD（Cloud Build + Artifact Registry + Cloud Deploy）

> [!abstract] 你會學到
> 平行化 build step、在 CI 裡用 **emulator 跑整合測試**、用 Secret Manager 傳祕密、**Cloud Deploy 的多環境晉升 + 核准 + canary + 回滾**。
> 對應筆記：[[Cloud Build]]、[[Artifact Registry]]、[[Cloud Deploy 與部署策略]]、[[Section 2 建置與測試應用]]

---

## 0️⃣ 準備

```bash
export PROJECT=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT --format='value(projectNumber)')
export REGION=asia-east1
gcloud services enable cloudbuild.googleapis.com artifactregistry.googleapis.com \
  clouddeploy.googleapis.com run.googleapis.com secretmanager.googleapis.com firestore.googleapis.com

gcloud artifacts repositories create lab04 --repository-format=docker --location=$REGION
```

---

## 1️⃣ 應用與測試

```bash
mkdir -p ~/pcd-lab04/tests && cd ~/pcd-lab04
```

`app.py`
```python
import os, json
from flask import Flask, jsonify, request
from google.cloud import firestore

app = Flask(__name__)
VERSION = os.environ.get("APP_VERSION", "v1")
db = firestore.Client()

@app.get("/healthz")
def healthz(): return "ok", 200

@app.get("/")
def root(): return jsonify(version=VERSION)

@app.post("/orders")
def create_order():
    body = request.get_json() or {}
    oid = body.get("id")
    if not oid:
        return jsonify(error="id required"), 400
    db.document(f"orders/{oid}").set({"id": oid, "total": body.get("total", 0)})
    return jsonify(id=oid), 201

@app.get("/orders/<oid>")
def get_order(oid):
    doc = db.document(f"orders/{oid}").get()
    return (jsonify(doc.to_dict()), 200) if doc.exists else (jsonify(error="not found"), 404)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

`tests/test_unit.py`
```python
def test_math():
    assert 1 + 1 == 2          # 代表你的純邏輯單元測試
```

`tests/test_integration.py`（需要 Firestore emulator）
```python
import os, pytest
os.environ.setdefault("FIRESTORE_EMULATOR_HOST", "localhost:8080")
os.environ.setdefault("GOOGLE_CLOUD_PROJECT", "demo-project")

from app import app as flask_app

@pytest.fixture
def client():
    flask_app.config["TESTING"] = True
    return flask_app.test_client()

def test_create_and_get(client):
    assert client.post("/orders", json={"id": "o-1", "total": 99}).status_code == 201
    r = client.get("/orders/o-1")
    assert r.status_code == 200 and r.get_json()["total"] == 99

def test_validation(client):
    assert client.post("/orders", json={}).status_code == 400
```

`requirements.txt`
```
Flask==3.0.3
gunicorn==22.0.0
google-cloud-firestore==2.19.0
pytest==8.3.3
```

`Dockerfile`（多階段 + 非 root）
```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
USER 1000
CMD ["gunicorn", "-b", "0.0.0.0:8080", "-w", "1", "--threads", "8", "app:app"]
```

---

## 2️⃣ `cloudbuild.yaml`（平行化 + emulator 整合測試 + 祕密）

```yaml
substitutions:
  _REGION: asia-east1
  _REPO: lab04
  _SERVICE: lab04-api

steps:
  # ① lint 與 unit test 平行跑（waitFor: ['-'] = 不等任何步驟）
  - id: lint
    name: python:3.12
    entrypoint: bash
    args: ['-c', 'pip install -q flake8 && flake8 --max-line-length=120 app.py || true']
    waitFor: ['-']

  - id: unit-test
    name: python:3.12
    entrypoint: bash
    args: ['-c', 'pip install -q -r requirements.txt && pytest -q tests/test_unit.py']
    waitFor: ['-']

  # ② 整合測試：起 Firestore emulator 再跑測試（Section 2.1 + 2.3 考點）
  - id: integration-test
    name: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
    entrypoint: bash
    args:
      - -c
      - |
        apt-get -qq update && apt-get -qq install -y python3-pip openjdk-17-jre-headless > /dev/null
        gcloud components install beta cloud-firestore-emulator --quiet
        gcloud beta emulators firestore start --host-port=localhost:8080 &
        for i in $(seq 30); do curl -s localhost:8080 >/dev/null && break; sleep 2; done
        pip3 install -q --break-system-packages -r requirements.txt
        FIRESTORE_EMULATOR_HOST=localhost:8080 GOOGLE_CLOUD_PROJECT=demo-project pytest -q tests/test_integration.py
    waitFor: ['unit-test']

  # ③ 建置（用 cache 加速）
  - id: build
    name: gcr.io/cloud-builders/docker
    args:
      - build
      - '--cache-from=${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:latest'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:$SHORT_SHA'
      - '-t'
      - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:latest'
      - '.'
    waitFor: ['integration-test', 'lint']

  # ④ 使用祕密（正確做法：availableSecrets + secretEnv）
  - id: use-secret
    name: gcr.io/cloud-builders/gcloud
    secretEnv: ['BUILD_TOKEN']
    entrypoint: bash
    args: ['-c', 'echo "token length: $${#BUILD_TOKEN}"']    # 只印長度，不印值
    waitFor: ['-']

availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/lab04-token/versions/latest
      env: BUILD_TOKEN

images:
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:$SHORT_SHA'
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_REPO}/${_SERVICE}:latest'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: E2_HIGHCPU_8
timeout: 1800s
```

```bash
echo -n "abc123secret" | gcloud secrets create lab04-token --data-file=-

# 給 Cloud Build 的 SA 讀祕密的權限
CB_SA=$PROJECT_NUMBER@cloudbuild.gserviceaccount.com
gcloud secrets add-iam-policy-binding lab04-token \
  --member="serviceAccount:$CB_SA" --role=roles/secretmanager.secretAccessor

git init -q && git add -A && git commit -q -m "lab04" || true
gcloud builds submit --config=cloudbuild.yaml --substitutions=SHORT_SHA=$(git rev-parse --short HEAD) .
```

> [!question] 觀察 ①
> 在 Cloud Build 的 UI（或 `gcloud builds log`）看步驟的時間軸：`lint`、`unit-test`、`use-secret` 是**同時開始**的嗎？
> 把 `waitFor: ['-']` 移除再跑一次，比較總時間 → 這就是平行化的價值。

---

## 3️⃣ Cloud Deploy：dev → prod（含核准與 canary）

```bash
# 先建兩個 Cloud Run 服務當 target（先部署一次讓服務存在）
IMG=$REGION-docker.pkg.dev/$PROJECT/lab04/lab04-api:latest
gcloud run deploy lab04-api-dev  --image $IMG --region $REGION --allow-unauthenticated --quiet
gcloud run deploy lab04-api-prod --image $IMG --region $REGION --allow-unauthenticated --quiet
```

`clouddeploy.yaml`
```yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: lab04-pipeline
serialPipeline:
  stages:
    - targetId: dev
    - targetId: prod
      strategy:
        canary:
          runtimeConfig:
            cloudRun:
              automaticTrafficControl: true
          canaryDeployment:
            percentages: [25, 50]
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
run:
  location: projects/PROJECT_ID/locations/REGION_ID
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
requireApproval: true            # ⭐ 生產需人工核准
run:
  location: projects/PROJECT_ID/locations/REGION_ID
```

`skaffold.yaml`
```yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata: { name: lab04 }
manifests:
  rawYaml: [run-service.yaml]
deploy:
  cloudrun: {}
```

`run-service.yaml`
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: lab04-api-dev
spec:
  template:
    spec:
      containers:
        - image: lab04-api
```

```bash
sed -e "s|PROJECT_ID|$PROJECT|g" -e "s|REGION_ID|$REGION|g" clouddeploy.yaml > cd.yaml
gcloud deploy apply --file=cd.yaml --region=$REGION --project=$PROJECT

# Cloud Deploy 的 SA 需要部署權限
CD_SA=$PROJECT_NUMBER-compute@developer.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$CD_SA" --role=roles/run.developer --quiet
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$CD_SA" --role=roles/clouddeploy.jobRunner --quiet

# 建立 release（綁定映像）
gcloud deploy releases create rel-$(date +%s) \
  --delivery-pipeline=lab04-pipeline --region=$REGION \
  --images=lab04-api=$IMG --skaffold-file=skaffold.yaml

# 觀察 rollout，晉升到 prod
gcloud deploy rollouts list --delivery-pipeline=lab04-pipeline --region=$REGION
gcloud deploy releases promote --release=REL_NAME --delivery-pipeline=lab04-pipeline --region=$REGION
# prod 需要核准
gcloud deploy rollouts approve ROLLOUT_NAME --release=REL_NAME \
  --delivery-pipeline=lab04-pipeline --region=$REGION
```

> [!question] 觀察 ②
> 晉升到 prod 時，**同一個映像**被部署，沒有重新建置 → 這就是 Cloud Deploy 與 Cloud Build 的分工：
> **CI 產生產出物一次，CD 把它晉升過各環境。**

**回滾**
```bash
gcloud deploy targets rollback prod --delivery-pipeline=lab04-pipeline --region=$REGION
```

---

## 4️⃣ 權限陷阱實驗（**高頻考點**）

```bash
# 建一個執行時 SA，讓 Cloud Build 用它部署 Cloud Run
gcloud iam service-accounts create lab04-runtime
RT=lab04-runtime@$PROJECT.iam.gserviceaccount.com

# 故意先不給 actAs，觀察錯誤
gcloud builds submit --no-source --config=<(cat <<EOF
steps:
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args: ['run','deploy','lab04-api-dev','--image','$IMG','--region','$REGION','--service-account','$RT','--quiet']
EOF
) 2>&1 | tail -5
# → 出現 iam.serviceaccounts.actAs 相關錯誤

# 補上權限後成功
gcloud iam service-accounts add-iam-policy-binding $RT \
  --member="serviceAccount:$CB_SA" --role=roles/iam.serviceAccountUser
```
> [!success] 學到什麼
> **Cloud Build 要用某個 SA 部署，必須對該 SA 有 `roles/iam.serviceAccountUser`（actAs）。**
> 這個錯誤訊息在實務與考試都很常見。

---

## ✅ 檢核清單

- [ ] 觀察到平行 step 同時開始，並比較有/無平行化的總時間
- [ ] 在 Cloud Build 裡成功用 **Firestore emulator** 跑整合測試
- [ ] 用 `availableSecrets` 傳祕密，且祕密值沒出現在 log 裡
- [ ] 建立 Cloud Deploy pipeline，完成 dev → prod 晉升
- [ ] prod 需要**人工核准**，並實際核准一次
- [ ] 觀察到 canary 25% → 50% 的流量變化
- [ ] 執行過一次 rollback
- [ ] 重現並修好 `actAs` 權限錯誤

## 🧹 清理

```bash
gcloud deploy delivery-pipelines delete lab04-pipeline --region=$REGION --force --quiet
gcloud run services delete lab04-api-dev lab04-api-prod --region=$REGION --quiet
gcloud secrets delete lab04-token --quiet
gcloud artifacts repositories delete lab04 --location=$REGION --quiet
gcloud iam service-accounts delete $RT --quiet
```

## 🔗 相關

- [[Cloud Build]]
- [[Artifact Registry]]
- [[Cloud Deploy 與部署策略]]
- [[開發環境 Cloud Code Shell Workstations 與 AI 工具]]
- [[Secret Manager 與 Cloud KMS]]
- [[IAM 與服務帳戶]]
- [[Cloud Run]]
- [[Section 2 建置與測試應用]]
- [[25 天衝刺計劃]]
