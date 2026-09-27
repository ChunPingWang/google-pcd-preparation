---
title: Google Professional Cloud Developer 備考 Vault
tags:
  - moc
  - gcp/pcd
updated: 2026-09-27
---

# 🎯 Google Cloud Professional Cloud Developer 備考 Vault

> [!abstract] 這是什麼
> 一份為 **Google Cloud Certified – Professional Cloud Developer (PCD)** 打造的 Obsidian 知識庫。
> 目標不只是「過考試」，而是讓你在真實專案裡能做出正確的技術決策：選對運算平台、設計對的資料模型、寫出可觀測且安全的服務。
>
> 內容依據 **2026 年版官方考試指南**（4 大範圍、含 AI 輔助開發與 Gen AI API 主題）撰寫。

---

## 🚀 三步開始

1. 先讀 [[00 考試總覽]] — 5 分鐘知道要考什麼、怎麼考。
2. 再讀 [[01 如何使用本 Vault]] 與 [[02 Obsidian 設定與外掛建議]] — 讓連結、閃卡、進度追蹤都能動。
3. 挑一份讀書計劃開始跑：[[25 天衝刺計劃]]（主推）或 [[替代方案 比較與選擇]]。

---

## 🗺️ Vault 地圖

| 資料夾 | 內容 | 什麼時候看 |
|---|---|---|
| `00-開始這裡` | 考試規格、資源清單、使用說明 | 第一天 |
| `01-讀書計劃` | 25 天 / 6 週 / 10 天三種節奏 + 每日追蹤 | 第一天，之後每天 |
| `02-考試範圍` | 官方 4 大 Section 的逐條拆解與對應筆記 | 當作「檢核清單」反覆回看 |
| `03-核心服務` | 每個服務的深度筆記（原理 + 限制 + 考點 + 真實場景） | 主要學習期 |
| `04-模式與決策` | 選型決策樹、設計模式、跨服務比較 | 學完服務後、考前必看 |
| `05-實作實驗室` | 6 個可在免費額度內跑完的動手練習 | 每個主題結束時 |
| `06-速查表` | gcloud / kubectl / 程式碼片段 / 數字限制 | 實作時查、考前背 |
| `07-練習題` | 情境題與詳解、誘答選項拆解 | 每個 Section 結束、考前兩天 |
| `08-Flashcards` | 間隔重複閃卡（`::` 格式） | 每天 10 分鐘 |

---

## 📚 四大考試範圍（含官方權重）

```mermaid
pie showData
    title PCD 考試權重
    "S1 設計可擴充/安全/可靠的雲端原生應用" : 32
    "S2 建置與測試應用" : 23
    "S3 設定雲端原生應用的部署" : 24
    "S4 整合 Google Cloud 服務" : 21
```

- [[Section 1 設計可擴充安全可靠的雲端原生應用]] — 32%｜**最重，先讀**
- [[Section 3 設定雲端原生應用的部署]] — 24%｜Cloud Run + GKE 實作細節
- [[Section 2 建置與測試應用]] — 23%｜開發環境、Cloud Build、測試、AI 工具
- [[Section 4 整合 Google Cloud 服務]] — 21%｜用戶端程式庫、可觀測性

---

## 🧭 最高頻考點捷徑

想快速補強最常出現的主題，直接跳這幾篇：

- [[決策樹 運算平台選型]] — 幾乎每份考卷都有 3～5 題靠這個判斷
- [[Cloud Run]] — 單一最重要的服務筆記
- [[決策樹 資料庫選型]] — Firestore / Spanner / Bigtable / Cloud SQL / AlloyDB 的分水嶺
- [[決策樹 訊息與事件選型]] — Pub/Sub vs Cloud Tasks vs Eventarc vs Workflows
- [[IAM 與服務帳戶]] + [[驗證與授權 ADC OAuth JWT]] — 安全題的地基
- [[韌性模式 重試 冪等 退避 斷路器]] — 「訊息重複處理」類題目的解法
- [[數字與限制速記]] — 考前 24 小時只讀這篇
- [[常見陷阱與誘答選項識別]] — 直接提升 5～10 分

---

## ✅ 進度儀表板

> [!tip] 需要 Dataview 外掛
> 若已安裝 Dataview，下面會自動列出所有筆記的複習狀態；沒安裝也不影響閱讀。

```dataview
TABLE status AS "狀態", confidence AS "信心(1-5)", file.mtime AS "最後修改"
FROM "03-核心服務"
SORT confidence ASC, file.name ASC
```

每篇服務筆記的 frontmatter 都有 `status` 與 `confidence` 欄位，讀完就更新：

```yaml
status: 未讀 | 讀過 | 熟練
confidence: 1   # 1=沒把握 5=可以教別人
```

---

## ⚠️ 關於資料時效

> [!warning] 以官方文件為準
> 本 Vault 中的配額、限制、預設值以 **2026-09-27** 查核為基準。Google Cloud 變動快（尤其 Cloud Run / GKE / Gen AI），考前請對照 [[03 官方資源清單]] 快速複查。
> 標記 🔢 的數字代表「可能被考、但也可能改版」。

## 🔗 相關

- [[00 考試總覽]]
- [[25 天衝刺計劃]]
- [[考前 48 小時衝刺]]
