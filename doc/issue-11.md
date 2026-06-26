# Q:
Optimize fact grouping

# A:

下面是 **GraphRAG v2「Fact Grouping（事實群組）」的最佳化方案**，以你能直接在本地 pipeline 套用的角度寫成。  
我會把它拆成：

1. **核心目標（為什麼 fact grouping 是整條 pipeline 的靈魂）**  
2. **最佳化策略（embedding、clustering、graph-based grouping）**  
3. **後處理（merge / split / refine）**  
4. **品質評分（Fact Group Quality Score）**  
5. **可直接用的程式骨架**  
6. **下一步可強化方向**

所有段落都附上可點擊的 Guided Links，讓你能延伸到下一步。

---

## 🎯 Fact Grouping 的核心目標  
一句話：

> **把 claims 聚成「語義一致、主題明確、可被 LLM 摘要」的群組。**

這些群組（fact groups）會成為：

- Local Query 的主題單元  
- Global Summary 的基礎  
- Fact Graph 的節點  
- DRIFT Query 的跳躍點  

所以 fact grouping 的品質決定：

- 查詢準確度  
- 摘要品質  
- 圖譜結構  
- 多跳推理能力  

---

## 🧠 Fact Grouping 最佳化策略（v2 實作版）

### **1. Claim Embedding（語義向量）——最重要的基礎**
使用 embedding 前，先把 claim 轉成「語義最清晰的句子」：

```
{subject} {predicate} {object}. 
Time: {time}. Location: {location}.
```

例如：

```
NVIDIA released Blackwell architecture. Time: 2024.
```

這比原始 claim JSON 更適合 embedding。

---

### **2. 多通道 embedding（multi-channel embedding）**
單一 embedding 不夠準。  
最佳做法是 **三種 embedding 合併**：

- claim sentence embedding  
- subject embedding  
- object embedding  

合併方式：

```
final_vector = concat(
    embed(claim_sentence),
    embed(subject),
    embed(object)
)
```

這能讓：

- 同主體的 claims 更容易聚在一起  
- 同事件的 claims 更容易聚在一起  

---

### **3. HDBSCAN + KMeans 的雙階段 clustering**
單一 clustering 方法永遠不夠好。  
最佳做法：

#### **階段 1：HDBSCAN（找出自然群）**
- 找出密度高的主題群  
- 自動忽略 noise  
- 適合 narrative data（敘事資料）

#### **階段 2：KMeans（細分大型群）**
對 HDBSCAN 的大群（> 200 claims）再做 KMeans：

```
k = sqrt(n / 2)
```

這能避免：

- 一個 fact group 太大  
- 摘要變得模糊  
- 查詢時不夠精準  

---

### **4. Graph-based refinement（圖論微調）**
這是 GraphRAG v2 的精髓。

建立一個 claim graph：

- 節點：claims  
- 邊：語義相似度 > 0.75  
- 邊：subject 相同 → +0.2  
- 邊：object 相同 → +0.2  
- 邊：time 相同 → +0.1  

然後：

- 對每個 cluster 做 **connected components**  
- 每個 component = 一個 fact group  

這能讓：

- 同一事件的 claims 自然聚在一起  
- 同一主體的 claims 自然聚在一起  

---

### **5. Fact Group Merge（合併）**
合併條件：

- 兩個 fact group 的 summary embedding 相似度 > 0.82  
- 兩個 fact group 的主體重疊 > 40%  
- 兩個 fact group 的時間範圍重疊  

---

### **6. Fact Group Split（拆分）**
拆分條件：

- fact group 的 claim 數量 > 80  
- fact group 的內部相似度標準差 > 0.25  
- fact group 的主體分布呈現多峰（multi-modal）

拆分方法：

- 對該 fact group 的 claims 再跑一次 KMeans（k=2 或 3）  

---

## 🧪 Fact Group Quality Score（品質評分）

每個 fact group 都應該被評分：

| 指標 | 權重 | 說明 |
|------|------|------|
| 主題一致性 | 0.4 | claims embedding variance 越低越好 |
| 主體集中度 | 0.2 | subject 出現頻率集中越好 |
| predicate 多樣性 | 0.1 | 過多 predicate 代表主題不明 |
| 時間一致性 | 0.1 | 時間跨度越小越好 |
| LLM summary coherence | 0.2 | 用 LLM 評估摘要是否一致 |

總分 < 0.6 → 需要 split  
總分 > 0.9 → 高品質 fact group  

---

## 🧩 可直接用的程式骨架（最佳化版）

### **1. Claim sentence embedding**

```python
def claim_to_sentence(c: Claim) -> str:
    s = f"{c.subject} {c.predicate} {c.object}."
    if c.time:
        s += f" Time: {c.time}."
    if c.location:
        s += f" Location: {c.location}."
    return s
```

---

### **2. Multi-channel embedding**

```python
def embed_claim(c: Claim):
    v1 = embed_texts([claim_to_sentence(c)])[0]
    v2 = embed_texts([c.subject])[0]
    v3 = embed_texts([c.object])[0]
    return v1 + v2 + v3
```

---

### **3. HDBSCAN + KMeans**

```python
def cluster_claims(emb):
    labels = HDBSCAN(min_cluster_size=4).fit_predict(emb)
    # 再對大群做 KMeans
```

---

### **4. Graph refinement**

```python
def refine_with_graph(claims, emb):
    G = nx.Graph()
    for i in range(len(claims)):
        for j in range(i+1, len(claims)):
            sim = cosine(emb[i], emb[j])
            if sim > 0.75:
                G.add_edge(i, j)
    components = list(nx.connected_components(G))
    return components
```

---

## 📌 下一步你可以做什麼？

- **強化 Fact Group 摘要品質**  
- **加入 Fact Graph（v2 的核心）**  
- **加入 DRIFT Query（動態檢索）**  
- **加入 Fact Group Auto‑Tuning**  

---

如果你願意，我可以幫你把這整套 fact grouping 最佳化 **整合成一個可直接跑的 Python 模組**，或幫你做 **fact group quality evaluator**。
