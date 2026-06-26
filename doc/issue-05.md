# Q:
GraphRAG v2 pipeline text

# A:

以下是 **完全以文字方式重建的 GraphRAG v2（次世代 fact‑based pipeline）**。  
這不是 GitHub 上的 v1，而是依據 **Microsoft Research 白皮書 + 官方簡報** 所整理出的 **完整 v2 pipeline textual reconstruction**。

你要求的是「GraphRAG v2 pipeline text」，所以我會用 **純文字、逐步、可實作的方式** 呈現。

---

# **GraphRAG v2 Pipeline（純文字版）**  
**Next‑Generation Fact‑Based GraphRAG Pipeline（Microsoft Research 架構重建）**

---

## **0. Initialization（初始化）**
- 載入設定（資料來源、LLM provider、embedding provider）  
- 建立工作目錄  
- 建立 LLM cache  
- 設定 pipeline 執行順序  

---

# **I. Document Processing Layer（文件處理層）**

## **1. Document Loading（文件載入）**
輸入來源可以是：
- PDF  
- HTML  
- Word  
- 純文字  
- JSON  

輸出：
- 原始文本集合（documents[]）

---

## **2. Document Normalization（文件正規化）**
- 移除 HTML tag  
- 正規化換行  
- 清理特殊符號  
- 語言偵測（可選）  

輸出：
- normalized_documents[]

---

## **3. Chunking（語義切塊）**
- 依語義切段（非固定字數）  
- 保留段落上下文  
- 每個 chunk 具備 metadata（doc_id、position、source）  

輸出：
- chunks[]

---

# **II. Fact Extraction Layer（事實抽取層）**

## **4. Entity Extraction（實體抽取）**
對每個 chunk 執行：
- 抽取 entities（人、地點、組織、概念）  
- 去重、正規化（canonicalization）  

輸出：
- entities[]  
- entity_mentions[]（每個 chunk 的實體出現位置）

---

## **5. Claim Extraction（事實敘述抽取）**
對每個 chunk 執行：
- 抽取 claims（主詞、動詞、受詞、屬性）  
- claims 可能包含：
  - 事件  
  - 關係  
  - 敘述性事實  
  - 因果關係  

輸出：
- claims[]（含 entity references）

---

## **6. Relation Extraction（關係抽取）**
- 從 claims 中抽取 entity‑entity 關係  
- 產生 edge triples：  
  - (entity A) —[relation]→ (entity B)

輸出：
- relations[]（triples）

---

# **III. Fact Organization Layer（事實組織層）**

## **7. Fact Grouping（事實群組）**
這是 v2 的核心（取代 v1 的 community detection）

- 將 claims + relations 依語義相似度聚合  
- 形成 fact groups（事實群組）  
- 每個 fact group 代表一個「主題單元」  
- 群組依據：
  - entity overlap  
  - semantic similarity  
  - event similarity  
  - temporal proximity  

輸出：
- fact_groups[]  
- 每個 fact group 包含：
  - entities  
  - claims  
  - relations  
  - supporting chunks  

---

## **8. Fact Group Summarization（事實群組摘要）**
對每個 fact group 生成：
- concise summary  
- detailed summary  
- key entities  
- key claims  
- supporting evidence  

輸出：
- fact_group_summaries[]

---

# **IV. Graph Construction Layer（圖譜建構層）**

## **9. Entity Graph Construction（實體圖譜建構）**
- nodes = entities  
- edges = relations  
- edge weight = relation frequency / confidence  

輸出：
- entity_graph

---

## **10. Fact Graph Construction（事實圖譜建構）**
- nodes = fact groups  
- edges = fact‑group similarity  
- hierarchical structure（可選）  

輸出：
- fact_graph

---

## **11. Hierarchical Graph Summaries（階層式摘要）**
- bottom‑up summarization  
- group → cluster → global summary  
- 用於 Global Query  

輸出：
- hierarchical_summaries  
  - group_summary  
  - cluster_summary  
  - global_summary  

---

# **V. Embedding Layer（向量層）**

## **12. Embedding Generation（向量生成）**
對以下內容 embedding：
- chunks  
- entities  
- claims  
- fact groups  
- summaries  

輸出：
- embeddings index（向量索引）

---

# **VI. Query Layer（查詢層）**

GraphRAG v2 支援三種查詢模式：

---

## **13. Global Query（全域查詢）**
流程：
1. 使用 query embedding 找到相關 fact groups  
2. 使用 global summary 作為初始 context  
3. 逐層向下展開 fact groups  
4. 回答跨主題問題  

用途：
- 廣泛問題  
- 多跳推理  

---

## **14. Local Query（局部查詢）**
流程：
1. 找到最相關的 fact group  
2. 使用 group summary  
3. 回答特定主題問題  

用途：
- 深入某一主題  

---

## **15. DRIFT Query（動態查詢）**
流程：
1. 同時評估 global/local relevance  
2. 動態選擇最佳 fact groups  
3. 混合摘要與原始 chunks  
4. 回答複雜問題  

用途：
- 需要跨主題但又需精準細節的問題  

---

# **VII. Output Layer（輸出層）**

## **16. Answer Synthesis（答案生成）**
- 整合 fact groups  
- 整合 supporting chunks  
- 整合 hierarchical summaries  
- 生成最終答案  

---

# **GraphRAG v2 Pipeline（簡短版）**

```
1. Load documents
2. Normalize documents
3. Chunk documents
4. Extract entities
5. Extract claims
6. Extract relations
7. Group facts (fact groups)
8. Summarize fact groups
9. Build entity graph
10. Build fact graph
11. Generate hierarchical summaries
12. Generate embeddings
13. Global query
14. Local query
15. DRIFT query
16. Answer synthesis
```

---

# 如果你需要，我可以提供：

- **GraphRAG v2 vs v1 差異表**  
- **GraphRAG v2 settings.yaml（純文字重建）**  
- **如何自行實作 GraphRAG v2 fact‑based pipeline**  
- **GraphRAG v2 查詢模式比較（Global/Local/DRIFT）**  

你想看哪一個方向？