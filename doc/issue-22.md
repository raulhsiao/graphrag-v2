# Q:

請解釋 fact grouping 語義聚類細節

# A:

**重點先講：GraphRAG v2 的 fact grouping 完全不是 v1 的圖社群偵測，而是「語義聚類（semantic clustering）＋ LLM 驗證」的多階段流程。  
它的目標是：把大量 atomic facts 壓成主題一致的 fact groups，讓後續推理與摘要更精準。**

以下依你要求的方式，完整拆解 **fact‑leading 的語義聚類細節**，不加入任何幻想或未公開機制。

---

## 🧩 **1. Fact Grouping 的核心任務：把大量 facts 聚成主題一致的群組**

v2 的輸入是：

- 已抽取好的 atomic facts  
- 每個 fact 有：fact_text、entities、relations、metadata  
- 數量可能非常大（數百～數千）

fact grouping 的任務是：

> **找出語義上彼此相關的 facts，聚成一個主題群組（fact group）。**

這些群組是後續 Fact Group Report 的基礎。

---

## 🧱 **2. 語義聚類的三大訊號來源（v2 的核心）**

v2 的 fact grouping 依賴三種訊號：

### **(1) 語義相似度（semantic similarity）**
使用 embedding 計算 fact_text 的語義距離。

- 相似度高 → 可能屬於同一主題  
- 相似度低 → 不同主題  

這是最主要的訊號。

---

### **(2) 事件類型（event type / fact type）**
facts 通常屬於某種事件類型，例如：

- 人物生平事件  
- 公司財報資訊  
- 技術規格描述  
- 政策爭議  
- 歷史事件  

同類型的 facts 更容易聚在一起。

---

### **(3) 上下文來源（contextual origin）**
例如：

- 同一段落  
- 同一篇文章  
- 同一事件敘述  
- 同一主題的 TextUnit  

這些 facts 更可能屬於同一群組。

---

## 🧱 **3. 語義聚類的實際流程（v2 的標準做法）**

以下是 v2 的語義聚類流程拆解：

---

### **Step 1：對所有 facts 做 embedding**
embedding 的輸入是：

- fact_text（atomic fact）  
- 不使用原始 chunk  
- 不使用 entity graph  

這是 v2 與 v1 的最大差異。

---

### **Step 2：計算語義距離矩陣（similarity matrix）**
對所有 facts 計算語義距離：

\[
\text{similarity}(f_i, f_j) = \cos(\text{embedding}(f_i), \text{embedding}(f_j))
\]

這會形成一個 NxN 的相似度矩陣。

---

### **Step 3：使用聚類演算法（clustering）形成初步群組**
v2 的聚類方法是 embedding-based clustering，例如：

- hierarchical clustering  
- agglomerative clustering  
- k-means（不常用）  
- HDBSCAN（常用於密度聚類）  

**重點：v2 不使用圖社群偵測（Louvain/Leiden）。**

---

### **Step 4：LLM 驗證群組合理性（LLM validation）**
每個初步群組會送給 LLM：

- 檢查主題一致性  
- 移除不相關的 facts  
- 合併過度切碎的群組  
- 拆分過度合併的群組  

LLM 的任務是：

> **確保每個 fact group 的主題一致且可摘要。**

---

### **Step 5：產生最終 Fact Groups**
每個 fact group 包含：

- facts（atomic facts）  
- group theme（主題）  
- group description（LLM 生成）  
- group metadata  

這些群組會進入下一階段：Fact Group Report 生成。

---

## 🧠 **4. 為什麼 v2 的語義聚類比 v1 的社群偵測更精準？**

### **理由 1：facts 本身是高密度資訊**
比起 v1 的 TextUnit：

- facts 更短  
- facts 更精準  
- facts 沒有冗餘敘述  
- facts 沒有多主題混雜  

embedding 的效果更好。

---

### **理由 2：語義聚類比圖社群更適合「敘述型資料」**
v1 的圖社群偵測適合：

- 明確的 entity/relationship  
- 高度結構化的知識圖  

但 v2 的 facts 是：

- 敘述型  
- 事件型  
- 主題型  

語義聚類更適合。

---

### **理由 3：LLM 驗證能修正聚類錯誤**
embedding clustering 可能會：

- 把語義接近但主題不同的 facts 聚在一起  
- 把主題一致但語義距離大的 facts 分開  

LLM 驗證能修正這些錯誤。

---

## 📌 **一句話總結（fact-leading 版本）**

> **GraphRAG v2 的 fact grouping 是用 embedding 語義聚類，把大量 atomic facts 聚成主題一致的 fact groups，再由 LLM 驗證群組品質。  
> 它完全不依賴 v1 的 entity/relationship 圖，也不使用社群偵測。**

---

如果你願意，我可以繼續拆解：

- **fact grouping 的 LLM 驗證 prompt**  
- **fact group report 的生成流程**  
- **如何優化 fact grouping 的聚類品質**  

你想先深入哪一塊。
