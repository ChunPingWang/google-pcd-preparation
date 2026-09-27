---
title: 02 Obsidian 設定與外掛建議
tags:
  - meta
  - obsidian
status: 讀過
confidence: 3
updated: 2026-09-27
---

# 02 Obsidian 設定與外掛建議

## ⚙️ 必要設定（沒設會有東西不顯示）

| 設定 | 位置 | 值 | 為什麼 |
|---|---|---|---|
| Wikilinks | Settings → Files & Links → **Use Wikilinks** | 開啟 | 本 Vault 全部用 `[[ ]]` 雙括號連結 |
| Default location for new notes | Files & Links | `Same folder as current file` | 新增筆記不會亂跑 |
| Strict line breaks | Editor | **關閉** | 表格與列表排版才正常 |
| Readable line length | Appearance | 開啟 | 長篇筆記好讀 |

## 🔌 外掛（社群外掛，皆非必要但強烈建議）

| 外掛 | 作用 | 在本 Vault 的用途 |
|---|---|---|
| **Spaced Repetition** | 間隔重複 | `08-Flashcards` 的 `::` 閃卡 |
| **Dataview** | 查詢筆記 | [[README]] 的進度儀表板、[[每日追蹤模板]] 的統計 |
| **Excalidraw** 或內建 **Canvas** | 畫圖 | Day 21 的「憑記憶畫架構圖」任務 |
| **Templater** | 模板 | 每日回顧、錯題卡 |
| **Advanced Tables** | 表格編輯 | 維護速查表不痛苦 |
| **Mermaid**（內建） | 流程圖 | 決策樹、架構圖已大量使用 |

> [!note] Mermaid 是內建的
> 本 Vault 的決策樹用 ` ```mermaid ` 語法，Obsidian 原生支援，不需外掛。若圖沒顯示，檢查程式碼區塊語言標記有沒有打錯。

---

## 🧩 建議的核心設定檔

如果你想直接套用，可以把下面內容寫進 `.obsidian/` 對應檔案（或手動在 UI 設定）。

**建議的 Spaced Repetition 設定**
```
Flashcard tags:        #flashcards
Single-line separator: ::
Multiline separator:   ?
Convert highlights to clozes: on
```

**錯題卡 Templater 模板**（放 `_templates/錯題卡.md`）
```markdown
---
tags: [weak, 錯題]
date: <% tp.date.now("YYYY-MM-DD") %>
section:
---
# 題目摘要


## 我選了什麼／為什麼錯


## 正確答案的判準（關鍵詞）


## 對應筆記
[[]]

## 一句話教訓
```

---

## 📱 跨裝置

- 用 **Obsidian Sync / iCloud / Git** 任一種同步都可以；本 Vault 是純 Markdown，沒有鎖定。
- 通勤時只讀 `08-Flashcards` 與 `06-速查表`（純文字、手機友善）。
- `05-實作實驗室` 需要終端機，安排在有電腦的時段。

## 🔗 相關

- [[01 如何使用本 Vault]]
- [[README]]
