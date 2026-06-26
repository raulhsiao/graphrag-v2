# Q:
Explain GraphRAG v2 architecture

# A:

以下內容以 **最新可查到的官方來源（Microsoft Research）** 為基礎，並依照你的偏好以 **繁體中文、技術精準、無圖片** 呈現。  
所有關鍵事實皆附上引用來源。

---

# **GraphRAG v2 架構（依官方 Microsoft Research 資訊整理）**

**核心結論：**  
GraphRAG v2 是一個 **端到端的資料管線（data pipeline）**，結合：

- LLM-based 結構化抽取  
- 知識圖譜（Knowledge Graph）  
- 社群偵測（Community Detection）  
- 多層級摘要（Hierarchical Summaries）  
- 多模式查詢（Global / Local / DRIFT）  

其目標是讓 LLM 能在大型文本資料中進行 **深層語義搜尋與推理**。  
官方來源明確指出 GraphRAG 是一個 **LLM-derived knowledge graph pipeline**   [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)。

---

# **一、GraphRAG v2 的整體架構流程**

GraphRAG v2 的 pipeline（官方稱為 *data pipeline and transformation suite*）包含以下階段：

---

## **1. 文件載入（Document Loading）**

來源可以是：

- PDF  
- HTML  
- Word  
- 純文字  

目標是將所有資料轉成可處理的文本。  
（官方描述 GraphRAG 是用來從非結構化文本中萃取結構化資料）  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/tools/?lang=zh-cn)。

---

## **2. Chunking（文本切塊）**

將文件切成語義一致的段落（chunks）。  
這些 chunks 後續會用於：

- graph extraction  
- embedding  
- 社群偵測  

---

## **3. Graph Extraction（圖譜抽取）**

GraphRAG 的核心：  
從每個 chunk 中抽取：

- **Entities（實體）**  
- **Relations（關係）**  
- **Claims（敘述性事實）**

這些資訊構成 **LLM-derived knowledge graph**（官方用語）  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)。

---

## **4. 社群偵測（Community Detection）**

GraphRAG 使用 network analysis 將知識圖譜分群。  
每個社群代表一個主題領域（topic cluster）。

官方文件指出 GraphRAG 結合：

- text extraction  
- network analysis  
- LLM summarization  

形成一個 end-to-end 系統  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)。

---

## **5. 社群摘要（Community Summaries）**

每個社群會生成：

- **Local Summary**（針對社群）  
- **Global Summary**（整體資料集）  

這些摘要是 GraphRAG 查詢的核心。

---

## **6. Embedding（向量化）**

GraphRAG 會對：

- chunks  
- entities  
- claims  

進行 embedding，用於後續查詢。

---

## **7. 查詢模式（Query Modes）**

GraphRAG v2 提供三種查詢模式（官方文件）：

### **(1) Global Query**  
從整體資料集的摘要開始，適合：

- 廣泛問題  
- 需要跨主題推理的問題  

### **(2) Local Query**  
從與 query 最相關的社群開始，適合：

- 特定主題  
- 深入某一領域  

### **(3) DRIFT Query**（2024 年 10 月發表）  
官方稱為 *Dynamic community selection*，能在 global/local 之間動態切換以提升品質與效率  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)。

---

# **二、GraphRAG v2 的核心技術理念**

根據 Microsoft Research 的官方描述，GraphRAG 的核心理念是：

> **結合文本抽取、網路分析與 LLM 摘要，形成一個可理解大型文本資料的系統。**  
> （官方原文：*combining text extraction, network analysis, and LLM prompting and summarization into a single end-to-end system*）  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)

這意味著：

- GraphRAG 不只是 RAG  
- 它是 **RAG + Knowledge Graph + Summarization + Network Analysis** 的混合體  
- 能進行 **multi-hop reasoning**（跨段落推理）

---

# **三、GraphRAG v2 與 v1 的差異（依官方資訊推論）**

雖然官方沒有明確標示「v1/v2」，但從 2024–2025 的更新可觀察到：

| 功能 | v1 | v2（依官方更新） |
|------|------|------|
| Knowledge Graph | 基本 | **LLM-derived KG（強化）** |
| 社群摘要 | 有 | **更完整的 global/local summaries** |
| 查詢模式 | Local / Global | **加入 DRIFT（動態切換）** |
| Pipeline | 基本 | **完整 data pipeline（官方強調）** |
| 自動調整 | 無 | **auto-tuning（2024/9 發表）** |

來源：GraphRAG auto-tuning、DRIFT Search、Global Search 改進等官方文章  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)。

---

# **四、GraphRAG v2 的官方來源（完整列表）**

以下是所有與 GraphRAG 架構相關的官方來源：

### **1. Microsoft Research – GraphRAG 主頁**  
包含架構、理念、新聞、技術更新  
  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)

### **2. Microsoft Research – GraphRAG Tools**  
描述 GraphRAG 是一個 data pipeline & transformation suite  
  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/tools/?lang=zh-cn)

### **3. Microsoft Research – GraphRAG 發表文章**  
包含 DRIFT、Global Search、Auto-tuning 等技術更新  
  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)

---

# **五、總結：GraphRAG v2 架構的本質**

GraphRAG v2 是一個：

- **以 LLM 為核心的知識圖譜建構系統**  
- **具備社群偵測與多層級摘要**  
- **支援多模式查詢（Global / Local / DRIFT）**  
- **能在大型文本資料中進行深層語義搜尋與推理**

其架構由 Microsoft Research 官方明確描述為：

> **LLM-derived knowledge graph + network analysis + summarization 的 end-to-end pipeline**  
  [Microsoft](https://www.microsoft.com/en-us/research/project/graphrag/?lang=zh-cn)

---

如果你需要，我可以進一步提供：

- **GraphRAG v2 的完整 pipeline 圖（純文字版）**  
- **GraphRAG v2 的 settings.yaml 最佳實踐**  
- **如何用 Neo4j/Kuzu 實作 GraphRAG v2**  
- **GraphRAG v2 的查詢模式比較（Global/Local/DRIFT）**

你想看哪一個？
