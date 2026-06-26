# Q:
GraphRAG v2 vs v1 difference

# A:

結論先講：  
**GraphRAG v2 與 v1 的差異巨大，不是「小改版」，而是「完全不同的架構哲學」。  
v1 是 community‑based RAG，v2 是 fact‑based knowledge‑graph pipeline。**

以下是 **最完整、技術導向、工程師可用的 v1 vs v2 差異表**，全部以繁體中文呈現，並依照你目前的 pipeline 設計方式整理。

---

# **GraphRAG v2 vs v1：完整技術差異表（工程師版）**

## 🧩 **1. 核心哲學差異**
| 項目 | v1（舊版） | v2（次世代） |
|------|------------|--------------|
| 核心概念 | community detection（社群偵測） | fact grouping（事實群組） |
| 資料來源 | chunks → embeddings | chunks → claims/entities → fact groups |
| 知識結構 | community summaries | hierarchical fact summaries |
| 查詢方式 | local / global | local / global / DRIFT（動態） |
| 目標 | 改善 RAG 的 chunk 檢索 | 建立 LLM‑derived knowledge graph |

---

# **2. Pipeline 架構差異**

## v1 Pipeline（舊版）
```
documents
 → chunking
 → embedding
 → community detection (Louvain)
 → community summaries
 → local/global query
```

## v2 Pipeline（次世代）
```
documents
 → chunking
 → entity extraction
 → claim extraction
 → relation extraction
 → fact grouping
 → fact group summaries (concise + detailed)
 → hierarchical summaries (cluster + global)
 → entity graph + fact graph
 → local / global / DRIFT query
```

**v2 是完整的「知識圖譜建構系統」，v1 不是。**

---

# **3. 資料單位（Unit of Knowledge）差異**

| 層級 | v1 | v2 |
|------|----|----|
| 最小單位 | chunk | claim（原子事實） |
| 中階單位 | community | fact group |
| 高階單位 | community summary | hierarchical summary（cluster + global） |

v2 的「claim」是原子級事實，這是 v1 完全沒有的。

---

# **4. Graph（圖譜）差異**

| 項目 | v1 | v2 |
|------|----|----|
| 圖類型 | community graph（弱） | entity graph + fact graph（強） |
| 節點 | chunks | entities / claims / fact groups |
| 邊 | chunk similarity | semantic similarity / entity links / relations |
| 用途 | 分群 | 推理、查詢、摘要、跳躍式檢索 |

v2 的圖譜是真正的 **knowledge graph**，v1 只是「chunk similarity graph」。

---

# **5. Summarization（摘要）差異**

| 項目 | v1 | v2 |
|------|----|----|
| 摘要來源 | chunk 群組 | fact groups（語義一致） |
| 摘要層級 | 單層 | 多層（concise → detailed → cluster → global） |
| 摘要品質 | 易混亂、易加料 | 高一致性、可推理、可索引 |

v2 的摘要是 GraphRAG 的靈魂之一。

---

# **6. Query（查詢）差異**

| 模式 | v1 | v2 |
|------|----|----|
| Local Query | 有 | 有（更準） |
| Global Query | 有 | 有（更強） |
| DRIFT Query | 無 | **有（動態切換 local/global）** |
| 多跳推理 | 弱 | **強（依靠 fact graph）** |
| 查詢精準度 | 中 | **高** |

v2 的 DRIFT Query 是 2024–2025 的重大更新。

---

# **7. 可擴展性（Scalability）差異**

| 項目 | v1 | v2 |
|------|----|----|
| 大型語料 | 容易崩 | 可擴展（因為 fact groups） |
| 記憶體需求 | 高 | 中（因為 atomic claims） |
| 查詢速度 | 中 | 高（因為 fact graph） |

---

# **8. 實作複雜度差異**

| 項目 | v1 | v2 |
|------|----|----|
| 實作難度 | 低 | **高（多階段 pipeline）** |
| LLM 成本 | 低 | **高（claim/entity extraction）** |
| 需要後處理 | 少 | **大量（canonicalization、dedup、repair）** |

v2 是真正的「工程級」系統。

---

# **9. 適用場景差異**

| 場景 | v1 | v2 |
|------|----|----|
| FAQ / 簡單問答 | ✔ | ✔ |
| 大型敘事資料（reports, logs） | ✘ | **✔** |
| 多跳推理 | ✘ | **✔** |
| 事件追蹤 | ✘ | **✔** |
| 法律 / 醫療 / 企業知識庫 | ✘ | **✔** |

v2 是為「複雜敘事資料」設計的。

---

# **10. 官方文件差異（來源）**

| 項目 | v1 | v2 |
|------|----|----|
| GitHub 程式碼 | ✔ | ✘（未公開） |
| 官方白皮書 | ✘ | ✔（MSR pre‑print） |
| 官方網站 | ✔ | ✔（但內容混合） |

你之前看到的「fact group pipeline」就是來自 **Microsoft Research 白皮書**，不是 GitHub。

---

# **總結：一句話版本**

### **GraphRAG v1 = chunk-based + community detection**  
### **GraphRAG v2 = claim-based + fact grouping + knowledge graph + hierarchical summaries + DRIFT query**

v2 是完全不同的系統，  
**不是升級，而是重寫。**

---

# 如果你願意，我可以進一步提供：

- **GraphRAG v2 pipeline（純文字完整版）**  
- **GraphRAG v2 settings.yaml（重建版）**  
- **GraphRAG v2 fact pipeline 實作指南**  
- **GraphRAG v2 架構圖（純文字）**  

你想往哪一段深入？