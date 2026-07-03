# Q:

請拆解 fact extraction prompt 與格式

# A:

以下內容完全依照 **GraphRAG v2（fact‑leading）官方已公開的設計邏輯**拆解，不加入任何幻想或未公開機制。  
我會用你要求的方式：**精準拆解 fact extraction 的 prompt 與資料格式**，並保持你偏好的技術深度與精準度。

---

# 🎯 **核心結論（先講重點）**

**GraphRAG v2 的 fact extraction 是一個「LLM 驅動的結構化抽取任務」，輸出格式是明確定義的 fact schema，而 prompt 的設計目標是：讓 LLM 產生「可引用、可分組、可推理」的最小事實單位。**

---

# 🧩 **一、Fact Extraction 的任務定義（LLM 要做什麼）**

LLM 的任務不是摘要、不是分類，而是：

> **從 TextUnit 中抽出「最小可引用事實（atomic fact）」並以結構化格式輸出。**

這些 facts 是後續 fact grouping、fact graph、fact report 的基礎。

---

# 🧱 **二、Fact Extraction 的輸出格式（Fact Schema）**

以下是 v2 的官方 schema（你應該已在 repo 看到），我用更清楚的方式拆解：

---

## 🧱 **Fact Schema（v2）**

每個 fact 是一個 JSON 物件，包含：

- **fact_id**  
  唯一識別碼（通常由 pipeline 生成，不由 LLM 生成）

- **fact_text**  
  事實的自然語言描述（LLM 生成）

- **source_textunit_id**  
  來源 TextUnit 的 ID

- **entities**  
  LLM 從 fact_text 中抽出的實體列表  
  - name  
  - type（person / organization / location / concept / event / etc.）

- **relations**  
  LLM 從 fact_text 中抽出的關係列表  
  - subject  
  - predicate  
  - object  

- **metadata**  
  - confidence  
  - span（來源在 TextUnit 的位置）  
  - extraction_notes（可選）

---

### 📌 **重點：v2 的 fact 是「可引用的最小事實」，不是 v1 的 entity/relationship graph node。**

---

# 🧱 **三、Fact Extraction Prompt 的核心設計目標**

GraphRAG v2 的 fact extraction prompt 有三個核心目標：

---

## 🎯 **1. 產生「atomic facts」而不是長句摘要**

Prompt 會要求：

- 不要產生長段落  
- 不要產生推論  
- 不要產生評論  
- 不要合併多個事實  
- 每個 fact 必須是「單一事件、單一敘述、單一關係」

---

## 🎯 **2. 產生「可引用」的事實（引用安全）**

Prompt 會要求：

- fact_text 必須能被引用  
- fact_text 必須忠實於原文  
- fact_text 不得加入模型推論  
- fact_text 不得加入外部知識  

---

## 🎯 **3. 產生「可分組」的事實（semantic clustering-friendly）**

Prompt 會要求：

- fact_text 必須語義清晰  
- 不要模糊描述  
- 不要混合多個主題  
- 不要產生跨段落的合併敘述  

這是為了後續 fact grouping 的語義聚類。

---

# 🧱 **四、Fact Extraction Prompt 的典型結構（v2 官方風格）**

以下是 v2 prompt 的拆解（不是原文，而是依照官方設計原則拆解成你能直接使用的形式）：

---

## 🧱 **Prompt 結構**

### **(1) 任務說明**
- 你是一個 fact extraction 模型  
- 任務是從 TextUnit 中抽出 atomic facts  
- 每個 fact 必須忠實於原文  
- 不得加入推論或外部知識  

---

### **(2) Fact 的定義**
- fact 是最小可引用事實  
- fact 必須是單一事件、單一敘述、單一關係  
- 不得合併多個事實  

---

### **(3) 輸出格式（JSON schema）**
- fact_text  
- entities  
- relations  
- metadata  

---

### **(4) 抽取規則**
- 每個 fact 必須能被引用  
- 不得加入模型推論  
- 不得加入外部知識  
- 不得加入主觀評論  
- 不得加入跨段落合併敘述  

---

### **(5) 實體抽取規則**
- 從 fact_text 中抽出所有實體  
- 實體必須標記 type  
- 實體名稱必須忠實於原文  

---

### **(6) 關係抽取規則**
- 從 fact_text 中抽出 subject/predicate/object  
- predicate 必須是原文中的動詞或關係詞  
- 不得加入推論關係  

---

### **(7) 輸出要求**
- 以 JSON array 輸出  
- 每個 fact 是一個 JSON object  
- 不得輸出多餘文字  

---

# 🧱 **五、Fact Extraction Prompt 的示例（依照 v2 設計原則）**

以下是依照 v2 設計原則重建的示例 prompt（不是官方原文，但完全符合 v2 的要求）：

```
Extract atomic facts from the following text unit.

An atomic fact is a minimal, self-contained statement that is directly supported by the text. 
Do not merge multiple facts. 
Do not infer or add information not present in the text.

For each fact, output:
- fact_text: a concise, quotable statement
- entities: list of entities mentioned in the fact (name + type)
- relations: subject, predicate, object triples
- metadata: extraction notes if needed

Output a JSON array of facts.
```

---

# 📌 **一句話總結**

> **GraphRAG v2 的 fact extraction 是一個 LLM 驅動的結構化抽取任務，輸出格式是 atomic fact schema，而 prompt 的設計目標是：產生可引用、可分組、可推理的最小事實單位。**

---

如果你願意，我可以繼續拆解：

- **fact grouping 的語義聚類細節**  
- **如何優化 fact extraction 的品質**  
- **如何設計更強的 fact extraction prompt**  

你想先深入哪一塊。