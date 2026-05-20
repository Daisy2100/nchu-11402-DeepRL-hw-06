# HW4 — AI Harness Systems Design
## 智慧學術研究助理（Smart Academic Research Assistant）

> AI Harness Systems Design and Analysis · 2026

---

## 專案簡介

本作業設計一套以 LLM 為核心控制器的 **智慧學術研究助理** AI Harness 系統。使用者輸入研究問題後，系統自動執行論文搜尋、摘要、比較與報告產出，全程由 LLM Orchestrator 透過 function calling 協調多個工具完成任務。

---

## 繳交檔案

| 檔案 | 說明 |
|------|------|
| `HW4_Report.docx` | 書面報告（系統設計、工具說明、workflow、evaluation） |
| `index.html` | 系統架構資訊圖表 (已部署至 GitHub Pages) |
| `HW4_log.md` | AI 輔助設計過程紀錄（prompt 歷史與設計決策） |
| `README.md` | 本說明文件 |

---

## 系統設計概覽

### 架構層次

```
[ 使用者研究問題 ]
        ↓
[ LLM Orchestrator ]  ↔  Working Memory（16k token）
        ↓ function calling      ↕
   [ Tool Layer ]         Research Cache（向量資料庫）
        ↓
[ 結構化研究報告 ]
```

### 工具清單（5 個）

| 工具 | 功能 |
|------|------|
| `search_papers()` | 跨資料庫搜尋學術論文（arXiv、Semantic Scholar） |
| `summarize_paper()` | 深度摘要單篇論文（方法 / 結果 / 限制） |
| `compare_literature()` | 比較多篇論文，產出比較矩陣 |
| `cite_formatter()` | 格式化引用清單（APA / IEEE / MLA） |
| `generate_report()` | 產出完整結構化文獻綜述報告 |

### Orchestration 模式

採用 **ReAct**（Reasoning + Acting）模式：

```
Thought → Action（Tool Call）→ Observation（Tool Result）→ 判斷是否完成 → 重複或輸出
```

---

## 如何檢視

1. **書面報告**：用 Microsoft Word 或 Google Docs 開啟 `HW4_Report.docx`
2. **網頁版資訊圖表**：直接用瀏覽器開啟 `index.html` 或訪問 [GitHub Pages 網址](https://Daisy2100.github.io/nchu-11402-DeepRL-hw-06/)
3. **設計紀錄**：任意 Markdown 閱讀器開啟 `HW4_log.md`

---

## 評量對應

| 評分項目 | 對應內容 |
|---------|---------|
| AI 系統設計完整性 | 報告第 2 節（架構）、第 3 節（工具）、第 6 節（orchestration） |
| Tool / Orchestration 設計 | 報告第 3 節工具 JSON Schema、第 6 節 ReAct 說明 |
| Workflow 與邏輯清晰度 | 報告第 4 節（7 步驟 workflow + 條件分支表格） |
| Infographic 視覺表達 | `index.html` |
| log.md 設計過程紀錄 | `HW4_log.md`（8 個迭代版本，含設計決策說明） |