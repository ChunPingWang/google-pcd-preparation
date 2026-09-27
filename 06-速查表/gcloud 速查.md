---
title: gcloud 速查
tags:
  - gcp/pcd
  - cheatsheet
status: 未讀
confidence: 1
updated: 2026-09-27
---

# gcloud 速查

> [!tip] 考試不會要你背參數，但會考「這個指令做了什麼」
> 重點是理解**每個旗標的語意**。實作時用這篇查，考前掃一遍加深印象。

---

## ⚙️ 基礎設定

```bash
gcloud init                                   # 互動式初始化
gcloud auth login                             # 使用者登入（gcloud CLI 用）
gcloud auth application-default login         # 設定 ADC（程式用）
gcloud auth application-default login --impersonate-service-account=SA   # ⭐ 模擬 SA 權限
gcloud auth list
gcloud auth print-access-token                # 呼叫 Google Cloud API 用
gcloud auth print-identity-token              # ⭐ 呼叫 Cloud Run / IAP 用

gcloud config set project PROJECT_ID
gcloud config set run/region asia-east1
gcloud config set compute/zone asia-east1-b
gcloud config list

# 多環境切換
gcloud config configurations create dev
gcloud config configurations activate dev
gcloud config configurations list

gcloud services enable run.googleapis.com pubsub.googleapis.com
gcloud services list --enabled
gcloud components update && gcloud components install beta kubectl skaffold
```

## 🎨 輸出格式（很實用）

```bash
gcloud run services list --format="table(metadata.name, status.url, status.traffic)"
gcloud run services describe api --format='value(status.url)'
gcloud projects list --format=json | jq -r '.[].projectId'
gcloud compute instances list --filter="status=RUNNING AND zone:asia-east1-b"
gcloud run services list --format="csv(metadata.name,status.url)"
```

---

## 🚀 Cloud Run

```bash
# 從原始碼部署（Buildpacks，不需 Dockerfile）
gcloud run deploy api --source . --region asia-east1

# 完整設定
gcloud run deploy api \
  --image asia-east1-docker.pkg.dev/$PROJECT/repo/api:v2 \
  --region asia-east1 \
  --service-account api-sa@$PROJECT.iam.gserviceaccount.com \
  --concurrency 40 --cpu 2 --memory 1Gi \
  --min-instances 1 --max-instances 50 --timeout 120 \
  --no-cpu-throttling \
  --set-env-vars "ENV=prod,FEATURE_X=on" \
  --set-secrets "DB_PASS=db-password:latest,/secrets/key=api-key:3" \
  --network default --subnet default --vpc-egress private-ranges-only \
  --add-cloudsql-instances $PROJECT:asia-east1:orders-db \
  --ingress internal-and-cloud-load-balancing \
  --no-allow-unauthenticated \
  --labels team=payments

# 流量管理
gcloud run deploy api --image IMG --no-traffic --tag canary
gcloud run services update-traffic api --to-tags canary=10
gcloud run services update-traffic api --to-latest
gcloud run services update-traffic api --to-revisions api-00007-abc=100   # 回滾

gcloud run revisions list --service api
gcloud run services update api --min-instances 0          # 改設定（產生新 revision）
gcloud run services add-iam-policy-binding api \
  --member="serviceAccount:caller@$PROJECT.iam.gserviceaccount.com" --role=roles/run.invoker

# Jobs
gcloud run jobs create etl --image IMG --tasks 20 --parallelism 5 \
  --task-timeout 30m --max-retries 3 --region asia-east1
gcloud run jobs execute etl --wait --region asia-east1
gcloud run jobs executions list --job etl --region asia-east1
```

---

## ☸️ GKE

```bash
gcloud container clusters create-auto prod --region asia-east1          # Autopilot ⭐
gcloud container clusters create std --zone asia-east1-a \
  --num-nodes 3 --machine-type e2-standard-4 --enable-ip-alias \
  --workload-pool=$PROJECT.svc.id.goog --enable-autoscaling --min-nodes 1 --max-nodes 10

gcloud container node-pools create spot --cluster std --zone asia-east1-a \
  --spot --enable-autoscaling --min-nodes 0 --max-nodes 20

gcloud container clusters get-credentials prod --region asia-east1
gcloud container clusters list
gcloud container clusters update prod --region asia-east1 --workload-pool=$PROJECT.svc.id.goog
```

---

## 🔐 IAM 與服務帳戶

```bash
gcloud iam service-accounts create api-sa --display-name="api runtime"
gcloud iam service-accounts list

# 專案層級授權
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:api-sa@$PROJECT.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"

# ⭐ 資源層級授權（範圍更小，考試偏好）
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:api-sa@$PROJECT.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
gcloud pubsub topics add-iam-policy-binding orders \
  --member="serviceAccount:api-sa@$PROJECT.iam.gserviceaccount.com" --role="roles/pubsub.publisher"
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:api-sa@$PROJECT.iam.gserviceaccount.com" --role="roles/storage.objectViewer"

# 允許 impersonation
gcloud iam service-accounts add-iam-policy-binding api-sa@$PROJECT.iam.gserviceaccount.com \
  --member="user:me@example.com" --role="roles/iam.serviceAccountTokenCreator"

# actAs（部署時指定 SA 需要）
gcloud iam service-accounts add-iam-policy-binding api-sa@$PROJECT.iam.gserviceaccount.com \
  --member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# 條件式存取
gcloud storage buckets add-iam-policy-binding gs://reports \
  --member="serviceAccount:$SA" --role="roles/storage.objectViewer" \
  --condition='expression=resource.name.startsWith("projects/_/buckets/reports/objects/public/"),title=public-only'

gcloud projects get-iam-policy $PROJECT --flatten="bindings[].members" \
  --filter="bindings.members:api-sa@$PROJECT.iam.gserviceaccount.com" --format="value(bindings.role)"
```

---

## 🗝 Secret Manager / KMS

```bash
echo -n "value" | gcloud secrets create db-password --data-file=- \
  --replication-policy=user-managed --locations=asia-east1
echo -n "new" | gcloud secrets versions add db-password --data-file=-
gcloud secrets versions access latest --secret=db-password
gcloud secrets versions list db-password
gcloud secrets versions disable 1 --secret=db-password
gcloud secrets update db-password --next-rotation-time="2026-12-01T00:00:00Z" --rotation-period="90d"

gcloud kms keyrings create app --location=asia-east1
gcloud kms keys create data-key --location=asia-east1 --keyring=app \
  --purpose=encryption --rotation-period=90d --next-rotation-time=...
gcloud kms encrypt --key=data-key --keyring=app --location=asia-east1 \
  --plaintext-file=in.txt --ciphertext-file=out.enc
```

---

## 📨 Pub/Sub / Tasks / Eventarc / Workflows / Scheduler

```bash
# Pub/Sub
gcloud pubsub topics create orders
gcloud pubsub subscriptions create orders-sub --topic=orders \
  --ack-deadline=30 --message-retention-duration=7d \
  --dead-letter-topic=orders-dlq --max-delivery-attempts=5 \
  --push-endpoint=https://svc-xxx.a.run.app/handle \
  --push-auth-service-account=pubsub-invoker@$PROJECT.iam.gserviceaccount.com
gcloud pubsub subscriptions create vip --topic=orders --message-filter='attributes.tier="vip"'
gcloud pubsub topics publish orders --message='{"id":1}' --attribute=tier=vip
gcloud pubsub subscriptions pull orders-sub --auto-ack --limit=10
gcloud pubsub subscriptions seek orders-sub --time=2026-09-26T00:00:00Z

# Cloud Tasks
gcloud tasks queues create email-q --location=asia-east1 \
  --max-dispatches-per-second=10 --max-concurrent-dispatches=5 --max-attempts=5
gcloud tasks queues pause email-q --location=asia-east1
gcloud tasks queues resume email-q --location=asia-east1

# Eventarc
gcloud eventarc triggers create img-uploaded --location=asia-east1 \
  --destination-run-service=thumbnailer --destination-run-region=asia-east1 \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-uploads" \
  --service-account=eventarc-sa@$PROJECT.iam.gserviceaccount.com
gcloud eventarc triggers list --location=asia-east1

# Workflows
gcloud workflows deploy process-order --source=workflow.yaml --location=asia-east1 \
  --service-account=wf-sa@$PROJECT.iam.gserviceaccount.com
gcloud workflows run process-order --data='{"orderId":"o-1"}' --location=asia-east1
gcloud workflows executions list process-order --location=asia-east1

# Scheduler
gcloud scheduler jobs create http nightly --location=asia-east1 \
  --schedule="0 2 * * *" --time-zone="Asia/Taipei" \
  --uri="https://svc-xxx.a.run.app/run" --http-method=POST \
  --oidc-service-account-email=sched-sa@$PROJECT.iam.gserviceaccount.com
gcloud scheduler jobs run nightly --location=asia-east1
```

---

## 🗄 資料與儲存

```bash
# Cloud Storage（新指令是 gcloud storage，舊的是 gsutil）
gcloud storage buckets create gs://my-bucket --location=asia-east1 --uniform-bucket-level-access
gcloud storage cp local.txt gs://my-bucket/
gcloud storage cp -r gs://my-bucket/dir ./            # 遞迴
gcloud storage ls -l gs://my-bucket/**
gcloud storage buckets update gs://my-bucket --lifecycle-file=lifecycle.json
gcloud storage buckets update gs://my-bucket --retention-period=7y
gcloud storage buckets update gs://my-bucket --lock-retention-period      # ⚠️ 不可逆
gcloud storage buckets update gs://my-bucket --versioning
gcloud storage sign-url gs://my-bucket/f.pdf --duration=15m \
  --impersonate-service-account=signer@$PROJECT.iam.gserviceaccount.com

# Cloud SQL
gcloud sql instances create orders-db --database-version=POSTGRES_16 \
  --tier=db-custom-2-7680 --region=asia-east1 --availability-type=REGIONAL \
  --no-assign-ip --network=projects/$PROJECT/global/networks/default \
  --enable-point-in-time-recovery
gcloud sql instances create orders-replica --master-instance-name=orders-db --region=asia-east1
gcloud sql users create app-sa@$PROJECT.iam --instance=orders-db --type=CLOUD_IAM_SERVICE_ACCOUNT
gcloud sql connect orders-db --user=postgres

# Firestore / Spanner / Bigtable
gcloud firestore databases create --location=asia-east1 --type=firestore-native
gcloud firestore export gs://backup-bucket/$(date +%F)
gcloud spanner instances create prod --config=regional-asia-east1 --processing-units=1000
gcloud bigtable instances create bt --display-name=bt \
  --cluster-config=id=c1,zone=asia-east1-b,nodes=3

# BigQuery
bq query --use_legacy_sql=false 'SELECT COUNT(*) FROM `p.d.t`'
bq load --source_format=PARQUET p:d.t gs://bucket/*.parquet
bq head -n 10 p:d.t                # ⭐ 免費預覽（不要用 SELECT * LIMIT 10）
bq show --schema p:d.t
```

---

## 🚚 建置與交付

```bash
gcloud artifacts repositories create apps --repository-format=docker --location=asia-east1
gcloud auth configure-docker asia-east1-docker.pkg.dev
gcloud artifacts docker images list asia-east1-docker.pkg.dev/$PROJECT/apps
gcloud artifacts docker images list-vulnerabilities IMAGE@sha256:...

gcloud builds submit --tag asia-east1-docker.pkg.dev/$PROJECT/apps/api:v1 .
gcloud builds submit --config=cloudbuild.yaml --substitutions=_ENV=prod .
gcloud builds list --limit=5
gcloud builds log BUILD_ID

gcloud builds triggers create github --name=main-deploy \
  --repo-owner=OWNER --repo-name=REPO --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml

gcloud deploy apply --file=clouddeploy.yaml --region=asia-east1
gcloud deploy releases create rel-$(git rev-parse --short HEAD) \
  --delivery-pipeline=api-pipeline --region=asia-east1 --images=api=IMG@sha256:...
gcloud deploy releases promote --release=REL --delivery-pipeline=api-pipeline --region=asia-east1
gcloud deploy rollouts approve ROLLOUT --release=REL --delivery-pipeline=api-pipeline --region=asia-east1
```

---

## 🔭 可觀測性

```bash
gcloud logging read 'resource.type="cloud_run_revision" AND severity>=ERROR' --limit=20 --freshness=1h
gcloud logging read "trace=\"projects/$PROJECT/traces/TRACE_ID\"" --limit=50    # ⭐ 撈整條鏈
gcloud logging tail 'resource.labels.service_name="api"'
gcloud logging metrics create payment_failures --log-filter='jsonPayload.error_code="CARD_DECLINED"'
gcloud logging sinks create archive storage.googleapis.com/projects/$PROJECT/buckets/log-archive \
  --log-filter='severity>=ERROR'
gcloud logging buckets update _Default --location=global --retention-days=90
gcloud logging sinks update _Default --add-exclusion=name=skip-health,filter='httpRequest.requestUrl:"/healthz"'
```

---

## 🌐 網路

```bash
gcloud compute network-endpoint-groups create run-neg --region=asia-east1 \
  --network-endpoint-type=serverless --cloud-run-service=api
gcloud compute backend-services create api-be --global \
  --load-balancing-scheme=EXTERNAL_MANAGED --enable-cdn
gcloud compute backend-services update api-be --global --session-affinity=GENERATED_COOKIE
gcloud compute url-maps invalidate-cdn-cache api-lb --path="/static/*"

gcloud compute security-policies create api-armor
gcloud compute security-policies rules create 1000 --security-policy=api-armor \
  --expression="evaluatePreconfiguredExpr('xss-v33-stable')" --action=deny-403
gcloud compute backend-services update api-be --global --security-policy=api-armor

gcloud compute networks vpc-access connectors create run-conn \
  --region=asia-east1 --network=default --range=10.8.0.0/28
gcloud compute routers nats create nat --router=rt --region=asia-east1 --auto-allocate-nat-external-ips
```

---

## 🧪 模擬器（Section 2 考點）

```bash
gcloud emulators firestore start --host-port=localhost:8080
gcloud emulators pubsub start --host-port=localhost:8085
gcloud emulators bigtable start
gcloud emulators spanner start
gcloud emulators datastore start
$(gcloud beta emulators pubsub env-init)          # 自動匯出環境變數
```

## 💰 帳單

```bash
gcloud billing budgets create --billing-account=$BA --display-name=lab \
  --budget-amount=10USD --threshold-rule=percent=50 --threshold-rule=percent=90
gcloud billing projects describe $PROJECT
```

## 🔗 相關

- [[kubectl 與 YAML 速查]]
- [[程式碼片段 Python Node Go]]
- [[數字與限制速記]]
- [[Cloud Run]]
- [[開發環境 Cloud Code Shell Workstations 與 AI 工具]]
