---
title: Lab 06 可觀測性 OpenTelemetry Trace 與 SLO
tags:
  - gcp/pcd
  - lab
  - service/opentelemetry
  - service/cloud-monitoring
status: 未做
confidence: 1
預估時間: 75 分鐘
updated: 2026-09-27
---

# Lab 06：可觀測性（OpenTelemetry + Trace 關聯 + SLO）

> [!abstract] 你會學到
> 建立**兩個服務的呼叫鏈**、用 OTel 傳播 trace、**讓 log 自動帶 trace ID**（一鍵撈出整條鏈的 log）、log-based metric、警示、SLO 與 burn rate、Error Reporting 聚合。
> 對應筆記：[[OpenTelemetry 與 Trace 關聯]]、[[Cloud Logging]]、[[Cloud Monitoring 與 SLO]]、[[Cloud Trace Profiler 與 Error Reporting]]

---

## 0️⃣ 準備

```bash
export PROJECT=$(gcloud config get-value project)
export REGION=asia-east1
gcloud services enable run.googleapis.com cloudtrace.googleapis.com \
  monitoring.googleapis.com logging.googleapis.com cloudprofiler.googleapis.com \
  clouderrorreporting.googleapis.com cloudbuild.googleapis.com

gcloud iam service-accounts create lab06-sa
SA=lab06-sa@$PROJECT.iam.gserviceaccount.com
for R in roles/logging.logWriter roles/cloudtrace.agent roles/monitoring.metricWriter \
         roles/cloudprofiler.agent roles/errorreporting.writer; do
  gcloud projects add-iam-policy-binding $PROJECT --member="serviceAccount:$SA" --role=$R --quiet
done
```

---

## 1️⃣ 共用的可觀測性模組

`obs.py`
```python
import json, os, sys, traceback
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

PROJECT = os.environ.get("GOOGLE_CLOUD_PROJECT") or os.environ.get("PROJECT_ID", "")
SERVICE = os.environ.get("SERVICE_NAME", "unknown")
VERSION = os.environ.get("SERVICE_VERSION", "1.0.0")

def init(app):
    resource = Resource.create({"service.name": SERVICE, "service.version": VERSION})
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(CloudTraceSpanExporter(project_id=PROJECT)))
    trace.set_tracer_provider(provider)
    FlaskInstrumentor().instrument_app(app)     # 自動為每個請求建 span
    RequestsInstrumentor().instrument()         # 自動為 outbound 呼叫建 span + 注入 traceparent
    try:
        import googlecloudprofiler
        googlecloudprofiler.start(service=SERVICE, service_version=VERSION, verbose=0)
    except Exception:
        pass

def _trace_fields():
    ctx = trace.get_current_span().get_span_context()
    if not ctx or not ctx.trace_id:
        return {}
    return {
        "logging.googleapis.com/trace": f"projects/{PROJECT}/traces/{format(ctx.trace_id, '032x')}",
        "logging.googleapis.com/spanId": format(ctx.span_id, "016x"),
        "logging.googleapis.com/trace_sampled": ctx.trace_flags.sampled,
    }

def log(severity, message, **fields):
    entry = {"severity": severity, "message": message,
             "serviceContext": {"service": SERVICE, "version": VERSION},
             **_trace_fields(), **fields}
    print(json.dumps(entry), file=sys.stdout, flush=True)

def log_exception(e, **fields):
    """整個 stack trace 放在同一個欄位 → Error Reporting 才能正確聚合"""
    entry = {"severity": "ERROR",
             "message": "".join(traceback.format_exception(type(e), e, e.__traceback__)),
             "serviceContext": {"service": SERVICE, "version": VERSION},
             **_trace_fields(), **fields}
    print(json.dumps(entry), file=sys.stderr, flush=True)

tracer = trace.get_tracer(__name__)
```

`requirements.txt`
```
Flask==3.0.3
gunicorn==22.0.0
requests==2.32.3
google-auth==2.35.0
google-cloud-profiler==4.1.0
opentelemetry-sdk==1.27.0
opentelemetry-exporter-gcp-trace==1.7.0
opentelemetry-instrumentation-flask==0.48b0
opentelemetry-instrumentation-requests==0.48b0
```
`Procfile`
```
web: gunicorn -b :$PORT -w 1 --threads 8 main:app
```

---

## 2️⃣ 下游服務（service-b）

```bash
mkdir -p ~/pcd-lab06/b && cd ~/pcd-lab06/b
cp ../obs.py . 2>/dev/null || true      # 或把 obs.py 複製進兩個目錄
```

`main.py`
```python
import os, random, time
from flask import Flask, jsonify
import obs
from obs import log, log_exception, tracer

app = Flask(__name__)
obs.init(app)

@app.get("/healthz")
def healthz(): return "ok", 200

@app.get("/price/<item>")
def price(item):
    with tracer.start_as_current_span("lookup_price") as span:
        span.set_attribute("item.id", item)          # 高基數放 span 屬性 ✅
        time.sleep(random.uniform(0.02, 0.25))       # 模擬資料庫查詢
        if item == "boom":
            try:
                raise ValueError(f"invalid item: {item}")
            except ValueError as e:
                log_exception(e, item=item)
                return jsonify(error="invalid item"), 500
        log("INFO", "price looked up", item=item)
        return jsonify(item=item, price=round(random.uniform(10, 100), 2))

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```bash
gcloud run deploy lab06-b --source . --region=$REGION --service-account=$SA \
  --set-env-vars "GOOGLE_CLOUD_PROJECT=$PROJECT,SERVICE_NAME=lab06-b,SERVICE_VERSION=1.0.0" \
  --allow-unauthenticated
B_URL=$(gcloud run services describe lab06-b --region=$REGION --format='value(status.url)')
```

---

## 3️⃣ 上游服務（service-a）呼叫 b

```bash
mkdir -p ~/pcd-lab06/a && cd ~/pcd-lab06/a
# 複製 obs.py、requirements.txt、Procfile
```

`main.py`
```python
import os, requests
from flask import Flask, jsonify
import obs
from obs import log, log_exception, tracer

app = Flask(__name__)
obs.init(app)
B_URL = os.environ["B_URL"]

@app.get("/healthz")
def healthz(): return "ok", 200

@app.get("/order/<item>")
def order(item):
    log("INFO", "order started", item=item)
    with tracer.start_as_current_span("validate_order"):
        pass
    try:
        # RequestsInstrumentor 會自動注入 traceparent → trace 跨服務串起來
        r = requests.get(f"{B_URL}/price/{item}", timeout=5)
        r.raise_for_status()
        price = r.json()["price"]
    except Exception as e:
        log_exception(e, item=item)
        return jsonify(error="pricing failed"), 502
    with tracer.start_as_current_span("charge_payment"):
        pass
    log("INFO", "order completed", item=item, price=price)
    return jsonify(item=item, price=price, status="ok")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

```bash
gcloud run deploy lab06-a --source . --region=$REGION --service-account=$SA \
  --set-env-vars "GOOGLE_CLOUD_PROJECT=$PROJECT,SERVICE_NAME=lab06-a,SERVICE_VERSION=1.0.0,B_URL=$B_URL" \
  --allow-unauthenticated
A_URL=$(gcloud run services describe lab06-a --region=$REGION --format='value(status.url)')

# 產生流量（含一些錯誤）
for i in $(seq 40); do curl -s -o /dev/null $A_URL/order/item-$((RANDOM % 5)); done
for i in $(seq 5);  do curl -s -o /dev/null $A_URL/order/boom; done
```

---

## 4️⃣ 驗證 trace 關聯（**本 Lab 的核心**）

```bash
# 取一筆 log 的 trace 欄位
TRACE=$(gcloud logging read 'jsonPayload.message="order completed"' --limit 1 \
  --format='value(trace)')
echo "TRACE=$TRACE"

# ⭐ 用 trace 一次撈出「兩個服務」的所有 log
gcloud logging read "trace=\"$TRACE\"" --limit 20 \
  --format="table(timestamp, resource.labels.service_name, severity, jsonPayload.message)"
```
> [!success] 檢核
> 你應該看到 **lab06-a 與 lab06-b 的 log 混在同一個 trace 裡**。
> 這就是官方考點「用 trace ID 關聯跨服務的 span」的實作證明。

```bash
# 在 Cloud Trace 看 span 瀑布圖
echo "https://console.cloud.google.com/traces/list?project=$PROJECT"
# 應該看到：lab06-a 的 span 底下嵌套 HTTP 呼叫 → lab06-b 的 span → lookup_price
```

---

## 5️⃣ Log-based metric + 警示

```bash
gcloud logging metrics create lab06_errors \
  --description="lab06 error logs" \
  --log-filter='resource.type="cloud_run_revision" AND (resource.labels.service_name="lab06-a" OR resource.labels.service_name="lab06-b") AND severity>=ERROR'

# 通知管道（email）
CH=$(gcloud beta monitoring channels create --display-name="lab06-email" \
  --type=email --channel-labels=email_address=$(gcloud config get-value account) \
  --format='value(name)')

cat > /tmp/alert.json <<EOF
{
  "displayName": "lab06 error rate high",
  "combiner": "OR",
  "conditions": [{
    "displayName": "errors > 1 per minute",
    "conditionThreshold": {
      "filter": "metric.type=\"logging.googleapis.com/user/lab06_errors\" AND resource.type=\"cloud_run_revision\"",
      "aggregations": [{"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE", "crossSeriesReducer": "REDUCE_SUM"}],
      "comparison": "COMPARISON_GT", "thresholdValue": 0.016, "duration": "60s"
    }
  }],
  "notificationChannels": ["$CH"],
  "documentation": {"content": "Runbook: 檢查 lab06-b 的 /price 端點", "mimeType": "text/markdown"}
}
EOF
gcloud alpha monitoring policies create --policy-from-file=/tmp/alert.json
```

---

## 6️⃣ Error Reporting

```bash
echo "https://console.cloud.google.com/errors?project=$PROJECT"
# 你應該看到「ValueError: invalid item: boom」被聚合成一個 issue，
# 顯示發生次數與首次出現的 service/version
```
> [!question] 實驗：破壞它
> 把 `log_exception` 改成 `print(f"error: {e}")`（單行、沒有 severity）再部署一次產生錯誤。
> → Error Reporting **抓不到**。這證明了四個必要條件（severity ≥ ERROR、完整 stack trace 在同一筆、結構化、不被吞掉）。

---

## 7️⃣ 建立 SLO 與 burn rate 概念

```bash
# Cloud Run 服務會自動出現在 Monitoring 的 Services 裡
echo "https://console.cloud.google.com/monitoring/services?project=$PROJECT"
```
在 UI 上為 `lab06-a` 建立 SLO：
1. **SLI**：Availability（`good = 2xx` / `total = all requests`）
2. **目標**：99% / rolling 7 天
3. 建立後看 **error budget** 剩餘百分比
4. 新增 **burn rate 警示**：`lookback 1h, threshold 10x`

```bash
# 製造錯誤消耗 error budget，觀察 budget 下降
for i in $(seq 30); do curl -s -o /dev/null $A_URL/order/boom; done
```
> [!success] 學到什麼
> Error budget 把「可靠性 vs 發布速度」變成一個**可協商的數字**。
> Burn rate 警示比固定門檻的假警報少得多 → 見 [[Cloud Monitoring 與 SLO]]。

---

## ✅ 檢核清單

- [ ] 兩個服務都送出 trace，且在 Cloud Trace 看到**嵌套的 span**（a → b）
- [ ] 用一個 `trace=` 查詢撈出**兩個服務**的 log
- [ ] 能說出 trace 關聯的三個必要條件
- [ ] Error Reporting 正確聚合例外，且能說明抓不到的四個原因
- [ ] 建立 log-based metric 與警示（含 runbook 文件）
- [ ] 建立 SLO，觀察 error budget 因錯誤而下降
- [ ] 能說明 burn rate 警示為何優於固定門檻
- [ ] 能說出「高基數資料放 span 屬性，不放 metric label」的理由

## 🧹 清理

```bash
gcloud run services delete lab06-a lab06-b --region=$REGION --quiet
gcloud logging metrics delete lab06_errors --quiet
gcloud alpha monitoring policies list --format='value(name)' \
  --filter='displayName:"lab06"' | xargs -r -n1 gcloud alpha monitoring policies delete --quiet
gcloud beta monitoring channels delete $CH --quiet
gcloud iam service-accounts delete $SA --quiet
```

## 🔗 相關

- [[OpenTelemetry 與 Trace 關聯]]
- [[Cloud Logging]]
- [[Cloud Monitoring 與 SLO]]
- [[Cloud Trace Profiler 與 Error Reporting]]
- [[Cloud Run]]
- [[韌性模式 重試 冪等 退避 斷路器]]
- [[Section 4 整合 Google Cloud 服務]]
- [[25 天衝刺計劃]]
