# HW4 設計過程紀錄 — log.md

**課程：** AI Harness Systems Design and Analysis  
**主題：** 智慧學術研究助理（Smart Academic Research Assistant）  
**紀錄日期：** 2026  

---

## 一、初始需求分析與場景選擇

### Prompt 1（需求釐清）

**User:**  
> 這份作業要我設計一個 AI Harness 系統，需要包含 LLM orchestrator、至少 3 個工具、workflow、evaluation 方法。你幫我建議一個好的場景。

**AI 回應摘要：**  
建議三個候選場景：
1. 智慧學術研究助理（搜尋論文 → 摘要 → 比較 → 報告）
2. 客服自動化系統（意圖分類 → 查詢 → 解答 → 升級）
3. 程式碼審查代理（靜態分析 → LLM 審查 → 建議 → PR 評論）

**設計決策：選擇場景 1（學術研究助理）**  
原因：
- 工具設計天然完整（搜尋、摘要、比較、格式化、報告）≥ 5 個工具
- Multi-step workflow 邏輯清晰（有明確的順序依賴性）
- Evaluation 指標好定義（ROUGE-L、Precision@k）
- 與學生日常使用情境相關，具說服力

---

## 二、系統架構設計迭代

### Prompt 2（初版架構）

**User:**  
> 幫我設計這個系統的 architecture，需要包含 LLM、tools、memory。

**AI 回應（初版）：**
```
User → LLM → Tools → Output
Memory: 只有 conversation history
工具：search, summarize, compare
```

**問題分析：**
- 記憶機制太簡單，缺乏長期記憶
- 只有 3 個工具，不夠完整
- 沒有考慮 error handling

---

### Prompt 3（架構修正）

**User:**  
> 這個架構記憶機制太弱了，而且缺少錯誤處理。幫我改進。

**AI 回應（第二版）：**  
新增雙層記憶機制：
- **Working Memory**：對話歷史 token window（16k）
- **Research Cache**：向量資料庫（Pinecone），儲存歷次摘要

新增錯誤處理：
- 搜尋失敗 → 切換備援資料庫
- 摘要幻覺偵測 → 與原文相似度比對
- 工具超時 → 10 秒 timeout + 部分結果回傳

**設計決策：採用第二版架構**  
新增 `cite_formatter` 和 `generate_report` 工具，使工具數達 5 個，覆蓋完整研究流程。

---

## 三、工具設計細節

### Prompt 4（工具 API 設計）

**User:**  
> 幫我用 JSON Schema 格式定義 5 個工具的 input/output。

**AI 回應（工具設計）：**

```json
{
  "name": "search_papers",
  "description": "跨資料庫搜尋學術論文",
  "parameters": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "搜尋關鍵字" },
      "year_range": { "type": "array", "items": { "type": "integer" } },
      "source": { "type": "string", "enum": ["arxiv", "semantic_scholar", "acm", "all"] },
      "max_results": { "type": "integer", "default": 10 }
    },
    "required": ["query"]
  }
}
```

類似格式為其他 4 個工具定義：`summarize_paper`、`compare_literature`、`cite_formatter`、`generate_report`

**架構調整：**  
討論後決定 `summarize_paper` 的 `focus_aspect` 參數改為 optional，讓 LLM 自行判斷摘要重點。原本設計為 required 參數，但這樣會讓 LLM 需要額外決策步驟。

---

## 四、Workflow 設計與邏輯

### Prompt 5（Workflow 流程）

**User:**  
> 幫我設計完整的 agent workflow，包含條件分支和錯誤處理。

**AI 回應：**  
設計 7 步驟 workflow，加入條件分支：
- 搜尋結果 < 5 篇 → 自動擴大搜尋
- 相關性分數 < 0.7 → 跳過該論文
- 論文 ≥ 2 篇 → 執行比較分析

**問題討論：**  
> 「如果使用者直接問比較性問題（如『哪些方法解決幻覺問題最有效』），應該直接跳到 compare_literature 嗎？」

**設計決策：**  
加入 Intent Parsing 步驟（Step 1），LLM 判斷：
- 探索型問題 → 完整流程（Step 1-7）
- 比較型問題 → 跳至 Step 4-7（略過逐篇摘要）
- 引用需求 → 直接 Step 7

---

## 五、Orchestration 機制

### Prompt 6（ReAct vs Plan-and-Execute）

**User:**  
> 這個系統應該用 ReAct 還是 Plan-and-Execute 模式？有什麼差別？

**AI 分析：**

| 面向 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 適用情境 | 動態、需即時反應 | 任務結構固定 |
| 複雜度 | 較低 | 較高 |
| 靈活性 | 高（可隨時調整） | 低（需提前規劃） |
| 可解釋性 | 高（每步都有 Thought） | 中 |

**設計決策：選用 ReAct 模式**  
學術研究問題的複雜度和所需工具呼叫次數往往不固定（有些問題只需 2 篇論文，有些需要 8 篇），ReAct 的動態性更適合。

---

## 六、Evaluation 方法設計

### Prompt 7（評估方法）

**User:**  
> 如何評估這個系統的效果？用什麼指標？

**AI 建議（初版）：**  
只建議了 ROUGE 和人工評分。

**User 追問：**  
> 「只有摘要品質不夠，還需要評估什麼？」

**AI 修正（第二版）：**  
完整評估框架：
1. **搜尋相關性**：Precision@5（目標 ≥ 0.80）
2. **摘要忠實度**：ROUGE-L + 人工評分（目標 ≥ 0.75）
3. **任務完成率**：End-to-end 測試集（目標 ≥ 90%）
4. **回應延遲**：P95 latency（目標 ≤ 30s）
5. **使用者滿意度**：CSAT 5 分制（目標 ≥ 4.0）

**新增評估流程設計：**
- Golden Test Set：50 個研究問題 + 人工標記理想輸出
- 自動化回歸測試（每次 prompt 修改後跑完整測試集）
- A/B Testing（比較不同 orchestration 策略）

---

## 七、設計問題與修正

### 問題 1：摘要幻覺問題

**發現：** AI 在生成摘要時可能產生幻覺（捏造論文結論）

**解決方案：**  
加入「幻覺偵測」機制：
- 比對生成摘要與原文 abstract 的相似度（cosine similarity ≥ 0.7）
- 低於閾值時重新摘要並提示 LLM 需「嚴格忠實於原文」

### 問題 2：Long-tail 搜尋效果差

**發現：** 對於小眾研究主題，單一資料庫搜尋結果可能不足

**解決方案：**  
備援機制：arXiv → Semantic Scholar → Google Scholar（fallback chain）

### 問題 3：並行工具呼叫

**討論：** Step 4 的 `summarize_paper` 需要對多篇論文並行呼叫，但 LLM context 可能不足以同時處理所有結果

**解決方案：**  
限制並行摘要最多 5 篇，超過部分以批次方式處理，並在 memory 中儲存中間結果

---

## 八、最終設計確認

### Prompt 8（最終確認）

**User:**  
> 幫我確認設計的完整性，有沒有遺漏？

**AI Checklist：**
- [x] AI 應用問題定義與使用情境
- [x] AI system architecture（LLM + tools + memory）
- [x] Function calling / tool usage 機制說明
- [x] 至少 3 個工具（已設計 5 個）
- [x] Agent workflow（多步驟任務執行流程）
- [x] Evaluation 方法
- [x] AI orchestration（ReAct 模式）

**補充優化：**  
LangGraph 對應說明：每個步驟為一個 Node，條件分支為 conditional edge，記憶透過 VectorStoreRetriever 整合

---

## 九、設計迭代總結

| 迭代版本 | 主要修改 | 原因 |
|---------|---------|------|
| v1 | 初版架構（3 工具，單層記憶） | 初始設計 |
| v2 | 新增雙層記憶，5 個工具 | 記憶機制不足 |
| v3 | 加入 Intent Parsing，動態跳轉流程 | 使用者場景多樣 |
| v4 | 幻覺偵測，備援搜尋機制 | 可靠性問題 |
| v5 | 完整評估框架（5 個指標） | 評估維度不足 |
| v6（最終）| 加入 LangGraph 對應說明，ReAct 模式確認 | 完整性確認 |

---

*本 log.md 記錄與 AI 工具合作完成系統設計的完整過程，包含初始構想、迭代修正與設計決策依據。*
