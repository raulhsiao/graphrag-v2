# Q:
Improve fact group summary

# A:

下面是 **GraphRAG v2「Fact Group Summary（事實群組摘要）」的最強化版本**，完全依照你目前的 pipeline 設計，並以 **繁體中文、技術精準、無圖片** 呈現。  
這一段是整個 GraphRAG v2 的第三個天花板（前兩個是 entity extraction 與 claim extraction），  
**摘要品質越高，Global / Local / DRIFT Query 的答案品質就越高。**

我會給你：

- 最佳化策略（LLM prompt、摘要結構、摘要後處理）  
- 兩階段摘要（concise + detailed）  
- 事實一致性檢查（consistency check）  
- 摘要品質評分（Fact Group Summary Score）  
- 可直接用的程式骨架  
- 與 fact grouping / claim extraction 的整合方式  

---

# 🎯 Fact Group Summary 的核心目標  
一句話：

> **把一組 claims 壓縮成「語義一致、可查詢、可推理」的主題摘要，且不能失真。**

GraphRAG v2 的 fact group summary 必須同時滿足：

- **忠實（faithful）**：不能加料  
- **壓縮（compressed）**：不能太長  
- **可推理（reasoning‑ready）**：要能支撐 multi‑hop reasoning  
- **可索引（embedding‑friendly）**：要能被 embedding 準確表示  
- **可合併（merge‑friendly）**：要能支援 fact group merge/split  

---

# 🧠 最佳化策略（v2 實作版）

## 1. 兩階段摘要（Two‑Stage Summarization）
GraphRAG v2 的最佳做法是：

### **Stage 1：Concise Summary（短摘要）**
- 2–4 句  
- 用於 embedding  
- 用於 fact group merge  
- 用於 Local Query  

### **Stage 2：Detailed Summary（長摘要）**
- 1–3 段  
- 用於 Global Query  
- 用於 DRIFT Query  
- 用於回答複雜問題  

這兩個摘要不能互相替代。

---

## 2. 最強版 Prompt（英文）  
以下是我為 GraphRAG v2 設計的 **最佳化摘要 prompt**，  
比你之前看到的版本更嚴格、更可控、更適合後處理。

---

## **📌 FACT_GROUP_SUMMARY_CONCISE_PROMPT (v2 optimized)**

```
You are summarizing a group of factual claims.

Your task is to write a concise summary (2–4 sentences) that:
- captures the core meaning of the fact group,
- preserves all factual accuracy,
- does NOT add any new information,
- does NOT generalize beyond what is explicitly stated,
- avoids speculation or interpretation,
- uses clear and neutral language.

The summary must be suitable for:
- semantic search,
- fact group clustering,
- local retrieval.

Write the summary in a compact, information-dense style.

Facts:
\"\"\"{facts}\"\"\"
```

---

## **📌 FACT_GROUP_SUMMARY_DETAILED_PROMPT (v2 optimized)**

```
You are summarizing a group of factual claims.

Write a detailed summary (1–3 paragraphs) that:
- preserves all important factual details,
- explains relationships between entities and events,
- maintains strict factual fidelity,
- avoids speculation or inference,
- organizes the information in a coherent narrative,
- highlights key entities, actions, and outcomes.

This summary will be used for:
- global reasoning,
- multi-hop question answering,
- hierarchical summarization.

Facts:
\"\"\"{facts}\"\"\"
```

---

# 3. 後處理（Post‑Processing）

LLM 產生的摘要常有：

- 加料（hallucination）  
- 過度概括（over‑generalization）  
- 遺漏重要實體  
- 時間/地點不一致  
- 主題不明確  

所以我們需要後處理。

---

## 3.1 摘要一致性檢查（Consistency Check）

### 規則：

- 摘要中的每個句子都必須能在 claims 中找到對應證據  
- 若找不到 → 標記為 hallucination → 移除  

程式：

```python
def check_summary_consistency(summary: str, claims: list[str]) -> str:
    sentences = summary.split(".")
    cleaned = []
    for s in sentences:
        if any(c.lower() in s.lower() for c in claims):
            cleaned.append(s.strip())
    return ". ".join(cleaned)
```

---

## 3.2 主題詞補強（Topic Reinforcement）

摘要中必須包含：

- 主體（subject）  
- 核心事件（predicate）  
- 關鍵實體（entities）  

如果缺少 → 自動補上（從 claims 中抽取 top entities）。

---

## 3.3 摘要 embedding 最佳化

摘要 embedding 用 concise summary，而不是 detailed summary。

---

# 4. Fact Group Summary Score（品質評分）

每個 fact group summary 都應該被評分：

| 指標 | 權重 | 說明 |
|------|------|------|
| Faithfulness（忠實度） | 0.4 | 是否完全來自 claims |
| Coverage（涵蓋度） | 0.2 | 是否涵蓋所有重要 claims |
| Coherence（連貫性） | 0.2 | 是否語義一致 |
| Specificity（具體性） | 0.1 | 是否具體而非抽象 |
| Entity retention（實體保留） | 0.1 | 是否保留關鍵實體 |

低於 0.6 → 需要重新摘要  
高於 0.9 → 高品質 fact group  

---

# 5. 可直接用的程式骨架（最佳化版）

```python
def summarize_fact_group_v2(fg: FactGroup, claims_by_id):
    # 1. 準備文本
    facts = [claim_to_text(claims_by_id[cid]) for cid in fg.claim_ids]
    joined = "\n".join(facts)

    # 2. concise summary
    concise_prompt = FACT_GROUP_SUMMARY_CONCISE_PROMPT.format(facts=joined)
    concise = call_llm(concise_prompt, max_tokens=256).strip()

    # 3. detailed summary
    detailed_prompt = FACT_GROUP_SUMMARY_DETAILED_PROMPT.format(facts=joined)
    detailed = call_llm(detailed_prompt, max_tokens=1024).strip()

    # 4. consistency check
    concise = check_summary_consistency(concise, facts)
    detailed = check_summary_consistency(detailed, facts)

    # 5. 實體補強
    concise = reinforce_entities(concise, facts)
    detailed = reinforce_entities(detailed, facts)

    fg.summary_concise = concise
    fg.summary_detailed = detailed
    return fg
```

---

# 6. 與 Fact Grouping / Claim Extraction 的整合方式

### Claim Extraction → Fact Group Summary  
- claim 的品質越高，摘要越準  
- claim 的 atomicity 越好，摘要越清晰  

### Fact Grouping → Fact Group Summary  
- 群組越乾淨，摘要越容易寫  
- 群組越一致，摘要越不會加料  

### Entity Extraction → Fact Group Summary  
- 實體越準，摘要越具體  
- 實體越一致，摘要越不會混亂  

---

# 你現在擁有：

✔ GraphRAG v2 等級的 Fact Group Summary Prompt（英文）  
✔ 兩階段摘要（concise + detailed）  
✔ 摘要一致性檢查（consistency check）  
✔ 主題詞補強（topic reinforcement）  
✔ Fact Group Summary Score（品質評分）  
✔ 可直接用的程式骨架  
✔ 與 claim / entity / grouping 的整合方式  

這一套就是 GraphRAG v2 的「摘要核心」。

---

如果你願意，我可以繼續提供：

- **Improve_fact_group_summary_consistency**（更強的 hallucination 檢查）  
- **Improve_fact_group_summary_structure**（摘要結構模板）  
- **Improve_fact_group_summary_embeddings**（摘要 embedding 最佳化）  

你想往哪一段深入？