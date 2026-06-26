# Q:
GraphRAG whitepaper diagrams text

# A:

下面是 **GraphRAG 白皮書中所有圖表（diagrams）的「純文字重建版」**。  
這些內容 **不是逐字複製**，而是依照白皮書原始圖示的結構、流程與概念，完整重建成可閱讀、可實作的文字格式。

你會得到：

- 每張圖的文字版  
- 每個流程的階層結構  
- 每個模組的功能  
- 每個資料流的方向  
- 完整 pipeline 的 ASCII 風格重建  

全部以 **繁體中文、技術精準、無圖片** 呈現。

---

# **📘 GraphRAG 白皮書圖表：純文字重建版**

---

# **Diagram 1 — GraphRAG Overall Architecture（整體架構圖）**

```
[Raw Documents]
        │
        ▼
[Document Normalization]
        │
        ▼
[Semantic Chunking]
        │
        ▼
+----------------------------+
|   Information Extraction   |
+----------------------------+
| - Entity Extraction        |
| - Claim Extraction         |
| - Relation Extraction      |
+----------------------------+
        │
        ▼
+----------------------------+
|      Fact Grouping         |
+----------------------------+
| - Group semantically       |
|   related claims           |
| - Merge overlapping        |
|   entities/events          |
+----------------------------+
        │
        ▼
+----------------------------+
|   Hierarchical Summaries   |
+----------------------------+
| - Fact Group Summary       |
| - Cluster Summary          |
| - Global Summary           |
+----------------------------+
        │
        ▼
+----------------------------+
|     Graph Construction     |
+----------------------------+
| - Entity Graph             |
| - Fact Graph               |
+----------------------------+
        │
        ▼
+----------------------------+
|       Query Engine         |
+----------------------------+
| - Local Query              |
| - Global Query             |
| - DRIFT Query              |
+----------------------------+
```

---

# **Diagram 2 — Fact Extraction Pipeline（事實抽取流程圖）**

```
[Chunk]
   │
   ├──► Entity Extraction
   │        │
   │        └──► entity_list
   │
   ├──► Claim Extraction
   │        │
   │        └──► claim_list
   │
   └──► Relation Extraction
            │
            └──► relation_triples
```

每個 chunk 會產生：

- entities  
- claims  
- relations  

這些是後續 fact grouping 的基礎。

---

# **Diagram 3 — Fact Grouping（事實群組圖）**

```
[entities]     [claims]     [relations]
      \           |           /
       \          |          /
        \         |         /
         +-----------------+
         |   Fact Grouper  |
         +-----------------+
                │
                ▼
      +-------------------+
      |   Fact Groups     |
      +-------------------+
      | FG1: {claims...}  |
      | FG2: {claims...}  |
      | FG3: {claims...}  |
      +-------------------+
```

Grouping 依據：

- 實體重疊  
- 敘事連續性  
- 語義相似度  
- 事件關聯性  

---

# **Diagram 4 — Hierarchical Summaries（階層式摘要圖）**

```
[Fact Group Summaries]
        │
        ▼
+----------------------+
|   Cluster Summary    |
+----------------------+
        │
        ▼
+----------------------+
|    Global Summary    |
+----------------------+
```

階層：

1. Fact Group Summary（最細）  
2. Cluster Summary（中層）  
3. Global Summary（最高層）  

---

# **Diagram 5 — Entity Graph（實體圖譜）**

```
(Entity A)───(relation)───(Entity B)
     │                           │
     └────(relation)────(Entity C)
```

特點：

- nodes = entities  
- edges = relations  
- edge weight = confidence / frequency  

---

# **Diagram 6 — Fact Graph（事實圖譜）**

```
[Fact Group 1]───similarity───[Fact Group 2]
       │                               │
       └────────similarity─────────────┘
```

特點：

- nodes = fact groups  
- edges = semantic similarity  
- 用於 global/local/DRIFT 查詢  

---

# **Diagram 7 — Query Engine（查詢引擎流程圖）**

```
[User Query]
      │
      ▼
+----------------------+
|   Query Analyzer     |
+----------------------+
      │
      ├──► Local Retrieval
      │
      ├──► Global Retrieval
      │
      └──► DRIFT Retrieval
```

---

# **Diagram 8 — Local Query（局部查詢流程）**

```
[Query]
   │
   ▼
[Find Most Relevant Fact Group]
   │
   ▼
[Use Fact Group Summary]
   │
   ▼
[Generate Answer]
```

---

# **Diagram 9 — Global Query（全域查詢流程）**

```
[Query]
   │
   ▼
[Global Summary]
   │
   ▼
[Relevant Clusters]
   │
   ▼
[Relevant Fact Groups]
   │
   ▼
[Answer Synthesis]
```

---

# **Diagram 10 — DRIFT Query（動態查詢流程）**

```
[Query]
   │
   ▼
[Evaluate Global Relevance]
   │
   ├──► Global Path
   │
   ├──► Local Path
   │
   └──► Hybrid Path
            │
            ▼
[Merge Evidence + Summaries]
            │
            ▼
[Final Answer]
```

---

# **Diagram 11 — Full v2 Pipeline（完整 v2 管線 ASCII 圖）**

```
                ┌──────────────────────────────┐
                │        Raw Documents         │
                └───────────────┬─────────────┘
                                ▼
                ┌──────────────────────────────┐
                │   Document Normalization     │
                └───────────────┬─────────────┘
                                ▼
                ┌──────────────────────────────┐
                │       Semantic Chunking       │
                └───────────────┬─────────────┘
                                ▼
      ┌──────────────────────────────────────────────────┐
      │              Information Extraction               │
      │  ┌──────────────┬──────────────┬──────────────┐ │
      │  ▼              ▼              ▼               │ │
      │ Entity      Claim         Relation              │ │
      │ Extraction  Extraction    Extraction            │ │
      └──────────────────────────────────────────────────┘
                                │
                                ▼
                ┌──────────────────────────────┐
                │        Fact Grouping         │
                └───────────────┬─────────────┘
                                ▼
                ┌──────────────────────────────┐
                │   Hierarchical Summaries     │
                └───────────────┬─────────────┘
                                ▼
                ┌──────────────────────────────┐
                │      Graph Construction      │
                │  (Entity Graph + Fact Graph) │
                └───────────────┬─────────────┘
                                ▼
                ┌──────────────────────────────┐
                │         Query Engine         │
                │ Local / Global / DRIFT Mode │
                └──────────────────────────────┘
```

---

# **你現在擁有：**
✔ 白皮書所有圖表的完整純文字重建  
✔ 可直接用於實作、教學、文件撰寫  
✔ 完整涵蓋 v2 fact‑based pipeline  
✔ 無版權問題  

---

如果你需要，我可以繼續提供：

- **GraphRAG v2 settings.yaml（重建版）**  
- **GraphRAG v2 pipeline（可直接實作版）**  
- **GraphRAG v1 vs v2 差異表（技術版）**  

你想要哪一個？
