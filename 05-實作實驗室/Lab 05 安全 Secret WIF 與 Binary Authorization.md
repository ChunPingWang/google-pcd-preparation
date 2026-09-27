---
title: Lab 05 安全 Secret WIF 與 Binary Authorization
tags:
  - gcp/pcd
  - lab
  - service/binary-authorization
  - service/iam
status: 未做
confidence: 1
預估時間: 75 分鐘
updated: 2026-09-27
---

# Lab 05：安全（Secret Manager + WIF + Binary Authorization）

> [!abstract] 你會學到
> 最小權限的實測（**故意觸發 403 再修好**）、Secret Manager 的版本與輪替、**Workload Identity Federation 讓 GitHub Actions 免金鑰部署**、Artifact Analysis 漏洞掃描、**Binary Authorization 擋下未簽署映像**。
> 對應筆記：[[IAM 與服務帳戶]]、[[Secret Manager 與 Cloud KMS]]、[[Workload Identity Federation]]、[[供應鏈安全 Artifact Analysis 與 Binary Authorization]]

---

## 0️⃣ 準備

```bash
export PROJECT=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT --format='value(projectNumber)')
export REGION=asia-east1
gcloud services enable secretmanager.googleapis.com cloudkms.googleapis.com \
  binaryauthorization.googleapis.com containeranalysis.googleapis.com \
  containerscanning.googleapis.com artifactregistry.googleapis.com run.googleapis.com \
  iamcredentials.googleapis.com sts.googleapis.com
```

---

## 1️⃣ 最小權限：故意做錯再修好

```bash
gcloud iam service-accounts create lab05-sa
SA=lab05-sa@$PROJECT.iam.gserviceaccount.com

echo -n "db-password-v1" | gcloud secrets create lab05-db --data-file=-

# 用 impersonation 模擬這個 SA 的身分（不下載金鑰！）
gcloud iam service-accounts add-iam-policy-binding $SA \
  --member="user:$(gcloud config get-value account)" \
  --role=roles/iam.serviceAccountTokenCreator

# ❌ 還沒授權 → 應該 403
gcloud secrets versions access latest --secret=lab05-db \
  --impersonate-service-account=$SA 2>&1 | tail -2

# ✅ 在「單一祕密」上授權（不是整個專案！）
gcloud secrets add-iam-policy-binding lab05-db \
  --member="serviceAccount:$SA" --role=roles/secretmanager.secretAccessor

sleep 10
gcloud secrets versions access latest --secret=lab05-db --impersonate-service-account=$SA
```
> [!success] 學到什麼
> 1. **403 PERMISSION_DENIED** = 身分對但缺角色（不是換 token 能解決的）→ 對應 [[Cloud API 呼叫最佳實務]] 的錯誤表。
> 2. **在資源層級授權**（單一 secret）比在專案層級授權範圍小得多 → 考試偏好這個答案。
> 3. **impersonation** 讓你不用下載金鑰就能測試 SA 的權限。

---

## 2️⃣ Secret 版本與輪替

```bash
# 新增版本
echo -n "db-password-v2" | gcloud secrets versions add lab05-db --data-file=-
gcloud secrets versions list lab05-db

gcloud secrets versions access latest --secret=lab05-db   # v2
gcloud secrets versions access 1      --secret=lab05-db   # v1（釘住版本）

# 停用舊版本（可逆），觀察行為
gcloud secrets versions disable 1 --secret=lab05-db
gcloud secrets versions access 1 --secret=lab05-db 2>&1 | tail -2   # 失敗
gcloud secrets versions enable 1 --secret=lab05-db

# 設定輪替通知
gcloud pubsub topics create lab05-rotation
gcloud pubsub topics add-iam-policy-binding lab05-rotation \
  --member="serviceAccount:service-$PROJECT_NUMBER@gcp-sa-secretmanager.iam.gserviceaccount.com" \
  --role=roles/pubsub.publisher
gcloud secrets update lab05-db \
  --next-rotation-time="$(date -u -v+1d '+%Y-%m-%dT%H:%M:%SZ' 2>/dev/null || date -u -d '+1 day' '+%Y-%m-%dT%H:%M:%SZ')" \
  --rotation-period="86400s" \
  --add-topics=projects/$PROJECT/topics/lab05-rotation
```

**在 Cloud Run 上比較 env vs volume**
```bash
# 用一個現成映像快速測試
cat > /tmp/secret-test.sh <<'EOF'
echo "ENV: $DB_PASS"
echo "FILE: $(cat /secrets/db 2>/dev/null)"
EOF

gcloud run deploy lab05-secret --image=gcr.io/google-samples/hello-app:1.0 \
  --region=$REGION --service-account=$SA \
  --set-secrets "DB_PASS=lab05-db:latest,/secrets/db=lab05-db:latest" \
  --no-allow-unauthenticated --quiet
```
> [!important] 觀念（不用真的跑完）
> - **env 注入**：只在**啟動時**解析 → 輪替後要重新部署才會拿到新值。
> - **volume 掛載 `latest`**：檔案內容可在不重啟的情況下更新（程式要會重讀）。
> 這是 [[Secret Manager 與 Cloud KMS]] 的高頻考點。

---

## 3️⃣ Workload Identity Federation（GitHub Actions 免金鑰）

```bash
gcloud iam workload-identity-pools create lab05-pool --location=global \
  --display-name="lab05 pool"

gcloud iam workload-identity-pools providers create-oidc lab05-github \
  --location=global --workload-identity-pool=lab05-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.ref=assertion.ref" \
  --attribute-condition="assertion.repository=='ChunPingWang/google-pcd-preparation' && assertion.ref=='refs/heads/main'"
  # ⭐ 沒有這個 condition = 任何 repo 都能拿到你的憑證（嚴重漏洞）

gcloud iam service-accounts create lab05-deployer
DEP=lab05-deployer@$PROJECT.iam.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$DEP" --role=roles/run.developer --quiet

POOL="projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/lab05-pool"
gcloud iam service-accounts add-iam-policy-binding $DEP \
  --role=roles/iam.workloadIdentityUser \
  --member="principalSet://iam.googleapis.com/$POOL/attribute.repository/ChunPingWang/google-pcd-preparation"

echo "Provider: $POOL/providers/lab05-github"
```

`.github/workflows/deploy.yml`（**放進你的 repo 才會實際執行**）
```yaml
name: deploy
on: { push: { branches: [main] } }
permissions:
  contents: read
  id-token: write                 # ⭐ 沒這行拿不到 OIDC token
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/lab05-pool/providers/lab05-github
          service_account: lab05-deployer@PROJECT.iam.gserviceaccount.com
      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud run services list --region asia-east1
```

> [!success] 學到什麼
> **整個流程沒有任何金鑰檔案**。`attribute-condition` 是安全的關鍵：限制只有指定 repo 的指定分支能換到憑證。
> 也順便禁止金鑰建立（組織政策）：`constraints/iam.disableServiceAccountKeyCreation`

---

## 4️⃣ Artifact Analysis：漏洞掃描

```bash
gcloud artifacts repositories create lab05 --repository-format=docker --location=$REGION
gcloud auth configure-docker $REGION-docker.pkg.dev --quiet

# 故意用一個舊的基底映像 → 會有一堆 CVE
cat > /tmp/Dockerfile.vuln <<'EOF'
FROM debian:10
CMD ["sleep", "3600"]
EOF
VULN=$REGION-docker.pkg.dev/$PROJECT/lab05/vuln:v1
docker build -t $VULN -f /tmp/Dockerfile.vuln /tmp && docker push $VULN

sleep 60
gcloud artifacts docker images list-vulnerabilities $VULN --format="value(name)" | head
gcloud artifacts docker images describe $VULN --show-package-vulnerability \
  --format="value(package_vulnerability_summary.vulnerabilities)" 2>/dev/null | head -5

# 對比：用 distroless 的乾淨映像
cat > /tmp/Dockerfile.clean <<'EOF'
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=busybox:1.36 /bin/busybox /busybox
ENTRYPOINT ["/busybox", "sleep", "3600"]
EOF
CLEAN=$REGION-docker.pkg.dev/$PROJECT/lab05/clean:v1
docker build -t $CLEAN -f /tmp/Dockerfile.clean /tmp && docker push $CLEAN
```
> [!success] 學到什麼
> 「映像有很多 CRITICAL CVE，最有效的長期解法」→ **換 minimal/distroless 基底 + 在 CI 加掃描 gate**，不是逐個修。

---

## 5️⃣ Binary Authorization：擋下未簽署的映像

```bash
# 建立簽章金鑰（非對稱）
gcloud kms keyrings create lab05-ring --location=$REGION
gcloud kms keys create lab05-key --location=$REGION --keyring=lab05-ring \
  --purpose=asymmetric-signing --default-algorithm=rsa-sign-pkcs1-4096

# 建立 note 與 attestor
cat > /tmp/note.json <<EOF
{ "name": "projects/$PROJECT/notes/lab05-note",
  "attestation": { "hint": { "human_readable_name": "lab05 build attestor" } } }
EOF
curl -s -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -d @/tmp/note.json \
  "https://containeranalysis.googleapis.com/v1/projects/$PROJECT/notes/?noteId=lab05-note" > /dev/null

gcloud container binauthz attestors create lab05-attestor \
  --attestation-authority-note=lab05-note --attestation-authority-note-project=$PROJECT
gcloud container binauthz attestors public-keys add --attestor=lab05-attestor \
  --keyversion-project=$PROJECT --keyversion-location=$REGION \
  --keyversion-keyring=lab05-ring --keyversion-key=lab05-key --keyversion=1

# 政策：預設要求 attestation（先用 DRYRUN 觀察，再切 ENFORCED）
cat > /tmp/policy.yaml <<EOF
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: DRYRUN_AUDIT_LOG_ONLY
  requireAttestationsBy:
    - projects/$PROJECT/attestors/lab05-attestor
globalPolicyEvaluationMode: ENABLE
EOF
gcloud container binauthz policy import /tmp/policy.yaml
```

**試著部署未簽署的映像**
```bash
DIGEST=$(gcloud artifacts docker images describe $CLEAN --format='value(image_summary.digest)')
IMG_DIGEST="$REGION-docker.pkg.dev/$PROJECT/lab05/clean@$DIGEST"

# DRYRUN 模式：允許但會記錄違規
gcloud run deploy lab05-binauthz --image=$IMG_DIGEST --region=$REGION \
  --binary-authorization=default --no-allow-unauthenticated --quiet
gcloud logging read 'protoPayload.serviceName="binaryauthorization.googleapis.com"' --limit 3 --freshness=10m

# 切成 ENFORCED
sed -i.bak 's/DRYRUN_AUDIT_LOG_ONLY/ENFORCED_BLOCK_AND_AUDIT_LOG/' /tmp/policy.yaml
gcloud container binauthz policy import /tmp/policy.yaml
sleep 20

# ❌ 現在應該被擋下來
gcloud run deploy lab05-binauthz --image=$IMG_DIGEST --region=$REGION \
  --binary-authorization=default --no-allow-unauthenticated --quiet 2>&1 | tail -3
```

**簽署後再部署**
```bash
gcloud container binauthz attestations sign-and-create \
  --artifact-url="$IMG_DIGEST" \
  --attestor=lab05-attestor --attestor-project=$PROJECT \
  --keyversion-project=$PROJECT --keyversion-location=$REGION \
  --keyversion-keyring=lab05-ring --keyversion-key=lab05-key --keyversion=1

sleep 15
gcloud run deploy lab05-binauthz --image=$IMG_DIGEST --region=$REGION \
  --binary-authorization=default --no-allow-unauthenticated --quiet   # ✅ 通過
```
> [!success] 學到什麼
> 1. **一定要先 DRYRUN** 再 ENFORCED，否則會擋掉既有部署。
> 2. **只能用 digest 部署**（tag 不行）→ 因為 tag 可變，digest 不可變。
> 3. 這就是「只有經過 CI 且通過檢查的映像才能上生產」的技術保證。

---

## ✅ 檢核清單

- [ ] 重現 403 並用**資源層級**授權修好
- [ ] 用 impersonation 測試 SA 權限（沒有下載金鑰）
- [ ] 操作過 secret 版本的 add / disable / 釘住版本
- [ ] 能說出 env 注入與 volume 掛載在輪替時的差異
- [ ] 建立 WIF pool + provider，且**設了 `attribute-condition`**
- [ ] 能說明沒有 condition 的風險
- [ ] 比較過舊基底映像與 distroless 的漏洞數量
- [ ] Binary Auth 在 DRYRUN 下記錄違規、在 ENFORCED 下擋下部署
- [ ] 簽署後成功部署，並理解為何必須用 digest

## 🧹 清理

```bash
cat > /tmp/policy-open.yaml <<'EOF'
defaultAdmissionRule:
  evaluationMode: ALWAYS_ALLOW
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
EOF
gcloud container binauthz policy import /tmp/policy-open.yaml
gcloud container binauthz attestors delete lab05-attestor --quiet
gcloud run services delete lab05-binauthz lab05-secret --region=$REGION --quiet
gcloud secrets delete lab05-db --quiet
gcloud pubsub topics delete lab05-rotation --quiet
gcloud artifacts repositories delete lab05 --location=$REGION --quiet
gcloud iam workload-identity-pools delete lab05-pool --location=global --quiet
gcloud iam service-accounts delete $SA --quiet
gcloud iam service-accounts delete $DEP --quiet
# KMS 金鑰無法刪除，只能停用版本（這是設計如此）
gcloud kms keys versions destroy 1 --key=lab05-key --keyring=lab05-ring --location=$REGION --quiet
```

## 🔗 相關

- [[IAM 與服務帳戶]]
- [[Secret Manager 與 Cloud KMS]]
- [[Workload Identity Federation]]
- [[供應鏈安全 Artifact Analysis 與 Binary Authorization]]
- [[Artifact Registry]]
- [[Cloud Build]]
- [[Cloud API 呼叫最佳實務]]
- [[Section 1 設計可擴充安全可靠的雲端原生應用]]
- [[25 天衝刺計劃]]
