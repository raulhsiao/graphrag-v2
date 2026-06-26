# Q:
GraphRAG v2 settings yaml text

# A:

我先講白：  
**目前沒有任何官方公開的「GraphRAG v2 settings.yaml」檔案。**  
下面這份是依你前面我們一起重建的 v2 pipeline，做出的 **合理、可實作的設定檔範本**——**不是官方原文，而是技術重建版**。

---

### 🧾 `graphrag_v2_settings.yaml`（重建版）

```yaml
# GraphRAG v2 – Fact-based pipeline settings (reconstructed)

project:
  name: graphrag_v2_fact_pipeline
  workspace_dir: ./workspace
  cache_dir: ./cache
  log_level: INFO

data:
  sources:
    - type: file
      path: ./data/docs
      patterns: ["*.pdf", "*.txt", "*.md"]
    - type: web
      urls:
        - "https://example.com/corpus1"
  language: "zh-TW"
  encoding: "utf-8"

document_normalization:
  enabled: true
  remove_html_tags: true
  normalize_whitespace: true
  normalize_unicode: true
  strip_control_chars: true

chunking:
  strategy: semantic
  max_tokens: 800
  min_tokens: 150
  overlap_tokens: 80
  respect_paragraphs: true
  use_headings_as_boundaries: true

llm:
  provider: openai
  model: gpt-4.1
  temperature: 0.2
  max_tokens: 2048
  retry:
    max_attempts: 3
    backoff_seconds: 2

entity_extraction:
  enabled: true
  prompt_template: "prompts/entity_extraction_zh.txt"
  max_entities_per_chunk: 50
  canonicalization:
    enabled: true
    strategy: "string_similarity+llm"
    similarity_threshold: 0.82

claim_extraction:
  enabled: true
  prompt_template: "prompts/claim_extraction_zh.txt"
  max_claims_per_chunk: 40
  include_temporal: true
  include_spatial: true
  include_attributes: true

relation_extraction:
  enabled: true
  prompt_template: "prompts/relation_extraction_zh.txt"
  max_relations_per_chunk: 60
  relation_types:
    - causal
    - temporal
    - part_of
    - member_of
    - associated_with

fact_grouping:
  enabled: true
  strategy: "semantic+entity_overlap"
  max_groups: 2000
  min_claims_per_group: 3
  similarity_metric: "cosine"
  similarity_threshold: 0.78
  use_temporal_proximity: true
  use_event_type: true

fact_group_summarization:
  enabled: true
  prompt_template_concise: "prompts/fact_group_summary_concise_zh.txt"
  prompt_template_detailed: "prompts/fact_group_summary_detailed_zh.txt"
  max_tokens_concise: 256
  max_tokens_detailed: 1024
  include_evidence_list: true

graph_construction:
  entity_graph:
    enabled: true
    edge_weight:
      type: "frequency+confidence"
      min_weight: 0.1
  fact_graph:
    enabled: true
    edge_weight:
      type: "semantic_similarity"
      min_weight: 0.2

hierarchical_summaries:
  enabled: true
  cluster_level:
    max_groups_per_cluster: 50
    similarity_threshold: 0.75
  global_level:
    max_clusters: 50
  prompt_template_cluster: "prompts/cluster_summary_zh.txt"
  prompt_template_global: "prompts/global_summary_zh.txt"

embeddings:
  provider: openai
  model: "text-embedding-3-large"
  index:
    type: "faiss"
    dim: 3072
  embed:
    chunks: true
    entities: true
    claims: true
    fact_groups: true
    summaries: true

query_engine:
  modes:
    - local
    - global
    - drift
  local:
    max_fact_groups: 20
    use_group_summary: true
    include_supporting_chunks: true
  global:
    max_clusters: 10
    max_fact_groups_per_cluster: 15
    use_global_summary: true
  drift:
    enabled: true
    global_weight: 0.6
    local_weight: 0.4
    max_fact_groups: 30

answer_synthesis:
  prompt_template: "prompts/answer_synthesis_zh.txt"
  max_tokens: 1024
  include_citations: true
  include_trace_explanation: true
```

---

這份可以當作你自己實作「fact‑based GraphRAG v2」的起點：  
- 想偏向 Neo4j / Kuzu 時，可以在 `graph_construction` 裡加 `backend: neo4j` / `kuzu`。  
- 想做 auto‑tuning，可以在 `query_engine` 加一個 `tuning` 區塊，專門記錄實驗參數。

如果你願意，我可以幫你把這份 settings 拆成「最小可跑版本」＋「進階選項」兩層，讓你在本地 LLM/向量庫環境直接套用。

# Implement GraphRAG v2 fact pipeline

我用「可以自己在本機實作」的角度來講，一句話總結：

> **GraphRAG v2 fact pipeline = 一條用 LLM 抽取 entity/claim → 用 embedding 分群成 fact groups → 建圖＋摘要 → 三種查詢模式的管線。**

下面我用「實作步驟 + 資料結構 + 關鍵程式片段」的方式給你。

---

### 一、技術棧建議

**語言 / 基礎：**

- **Python 3.10+**
- **向量庫：FAISS / Qdrant / Chroma 任一**
- **圖庫：Neo4j / NetworkX / Kùzu 任一**
- **LLM：你本機或雲端的（OpenAI / local LLM）**

---

### 二、資料結構設計（最小可用版）

**1. Chunk**

```python
class Chunk(BaseModel):
    id: str
    doc_id: str
    text: str
    order: int
```

**2. Entity**

```python
class Entity(BaseModel):
    id: str
    name: str
    type: str  # PERSON / ORG / PLACE / EVENT / OTHER
```

**3. Claim**

```python
class Claim(BaseModel):
    id: str
    chunk_id: str
    subject: str
    predicate: str
    object: str
    time: str | None = None
    location: str | None = None
    extra: dict = {}
```

**4. Relation（三元組）**

```python
class Relation(BaseModel):
    id: str
    head_entity_id: str
    tail_entity_id: str
    relation_type: str
    source_claim_id: str
```

**5. FactGroup**

```python
class FactGroup(BaseModel):
    id: str
    claim_ids: list[str]
    entity_ids: list[str]
    summary_concise: str | None = None
    summary_detailed: str | None = None
```

---

### 三、Pipeline 步驟實作順序

#### 1. 載入與切塊

```python
def load_documents(path: str) -> list[str]:
    # 你可以自己處理 PDF/TXT，這裡假設已轉成純文字
    ...

def chunk_document(text: str, max_tokens=800, overlap=80) -> list[Chunk]:
    # 用 tiktoken / 自己的 tokenizer 切語義 chunk
    ...
```

---

#### 2. Entity / Claim / Relation 抽取（LLM）

你可以用一個通用的 `call_llm(prompt) -> str` 包起來。

**(1) Entity 抽取**

Prompt 大概長這樣（示意）：

```text
你是一個資訊抽取系統，請從以下文本中抽取實體：
- 人物
- 組織
- 地點
- 事件
請輸出 JSON 陣列，每個元素包含：
{name, type}

文本：
{{chunk_text}}
```

程式：

```python
def extract_entities(chunk: Chunk) -> list[Entity]:
    prompt = render_entity_prompt(chunk.text)
    resp = call_llm(prompt)
    data = json.loads(resp)
    return [
        Entity(id=uuid4().hex, name=e["name"], type=e["type"])
        for e in data
    ]
```

**(2) Claim 抽取**

```text
請將以下文本拆解為「事實敘述（claims）」。
每個 claim 請輸出：
- subject
- predicate
- object
- time (若有)
- location (若有)
輸出 JSON 陣列。
```

```python
def extract_claims(chunk: Chunk) -> list[Claim]:
    prompt = render_claim_prompt(chunk.text)
    resp = call_llm(prompt)
    data = json.loads(resp)
    return [
        Claim(
            id=uuid4().hex,
            chunk_id=chunk.id,
            subject=c["subject"],
            predicate=c["predicate"],
            object=c["object"],
            time=c.get("time"),
            location=c.get("location"),
        )
        for c in data
    ]
```

**(3) Relation 抽取**

可以直接從 claim 推：

```python
def derive_relations_from_claim(claim: Claim, entity_index: dict[str, Entity]) -> list[Relation]:
    # 用簡單規則：subject/object 在 entity_index 裡就連邊
    ...
```

---

### 四、Fact Grouping（用 embedding + clustering）

**1. 先把每個 claim 轉成一句描述**

```python
def claim_to_text(claim: Claim) -> str:
    parts = [claim.subject, claim.predicate, claim.object]
    if claim.time:
        parts.append(f"時間：{claim.time}")
    if claim.location:
        parts.append(f"地點：{claim.location}")
    return "；".join(parts)
```

**2. 對所有 claim 做 embedding**

```python
def embed_claims(claims: list[Claim]) -> np.ndarray:
    texts = [claim_to_text(c) for c in claims]
    return embedding_model.embed(texts)  # shape: (N, D)
```

**3. 用 clustering 做 fact groups（最簡單：KMeans / HDBSCAN）**

```python
from sklearn.cluster import HDBSCAN

def group_claims(claims: list[Claim], embeddings: np.ndarray) -> list[FactGroup]:
    clusterer = HDBSCAN(min_cluster_size=3, metric="euclidean")
    labels = clusterer.fit_predict(embeddings)

    groups: dict[int, list[Claim]] = {}
    for label, claim in zip(labels, claims):
        if label == -1:
            continue  # noise
        groups.setdefault(label, []).append(claim)

    fact_groups: list[FactGroup] = []
    for gid, group_claims in groups.items():
        claim_ids = [c.id for c in group_claims]
        # 這裡可以再從 claim 裡收集 entity_ids
        fact_groups.append(
            FactGroup(
                id=f"fg_{gid}",
                claim_ids=claim_ids,
                entity_ids=[],
            )
        )
    return fact_groups
```

---

### 五、Fact Group 摘要（LLM）

```python
def summarize_fact_group(fg: FactGroup, claims_by_id: dict[str, Claim]) -> FactGroup:
    texts = [claim_to_text(claims_by_id[cid]) for cid in fg.claim_ids]
    joined = "\n".join(texts)
    prompt_concise = render_fg_summary_prompt(joined, mode="concise")
    prompt_detailed = render_fg_summary_prompt(joined, mode="detailed")

    concise = call_llm(prompt_concise)
    detailed = call_llm(prompt_detailed)

    fg.summary_concise = concise.strip()
    fg.summary_detailed = detailed.strip()
    return fg
```

---

### 六、Graph 建構（Entity Graph + Fact Graph）

**1. Entity Graph（用 NetworkX 示意）**

```python
import networkx as nx

def build_entity_graph(entities: list[Entity], relations: list[Relation]) -> nx.Graph:
    G = nx.Graph()
    for e in entities:
        G.add_node(e.id, name=e.name, type=e.type)
    for r in relations:
        G.add_edge(r.head_entity_id, r.tail_entity_id, relation_type=r.relation_type)
    return G
```

**2. Fact Graph（以 fact group embedding + similarity 建邊）**

```python
def build_fact_graph(fact_groups: list[FactGroup], fg_embeddings: np.ndarray, threshold=0.75) -> nx.Graph:
    G = nx.Graph()
    for fg in fact_groups:
        G.add_node(fg.id)

    # 簡單 O(N^2) 相似度（實務上可用 FAISS）
    from sklearn.metrics.pairwise import cosine_similarity
    sim = cosine_similarity(fg_embeddings)

    n = len(fact_groups)
    for i in range(n):
        for j in range(i + 1, n):
            if sim[i, j] >= threshold:
                G.add_edge(fact_groups[i].id, fact_groups[j].id, weight=float(sim[i, j]))
    return G
```

---

### 七、查詢引擎（Local / Global / DRIFT）

**1. Local Query（找最相關 fact groups）**

```python
def local_query(query: str, fg_embeddings: np.ndarray, fact_groups: list[FactGroup], top_k=10):
    q_emb = embedding_model.embed([query])[0]
    scores = fg_embeddings @ q_emb
    idx = np.argsort(-scores)[:top_k]
    return [fact_groups[i] for i in idx]
```

**2. Global Query（先用 global summary / cluster，再下去找 fact groups）**

最小版可以簡化成：

- 直接在所有 fact group 上做檢索（其實就是 local 的全域版）  
- 或另外對「cluster summary / global summary」做一層 embedding 檢索再展開  

**3. DRIFT Query（混合策略）**

簡化版可以：

- 同時計算：
  - 對 fact group 的相似度（local）
  - 對 global summary 的相似度（global）
- 用權重合併分數，取前 k 個 fact groups

---

### 八、答案生成（Answer Synthesis）

```python
def answer_query(query: str, selected_fact_groups: list[FactGroup], claims_by_id: dict[str, Claim]) -> str:
    context_blocks = []
    for fg in selected_fact_groups:
        context_blocks.append(f"【Fact Group】{fg.summary_concise}")
        for cid in fg.claim_ids[:5]:
            context_blocks.append(f"- {claim_to_text(claims_by_id[cid])}")
    context = "\n".join(context_blocks)

    prompt = f"""
你是一個知識圖譜輔助的問答系統。
根據以下事實群組與敘述，回答使用者問題。

問題：
{query}

事實與摘要：
{context}

請用條列且精簡的方式回答。
"""
    return call_llm(prompt).strip()
```

---

### 九、最小可跑版本的實作順序建議

1. **先做：**  
   - chunking  
   - claim 抽取  
   - claim embedding + clustering → fact groups  
   - fact group 摘要  
   - local query + answer synthesis  

2. **再加：**  
   - entity / relation 抽取  
   - entity graph / fact graph  
   - global / DRIFT 模式  

3. **最後才優化：**  
   - LLM cache  
   - pipeline 併行  
   - auto‑tuning / 評估  

---

如果你願意，我可以下一步幫你做其中一件事：

- 把這整條 pipeline 收斂成一個 `run_pipeline.py` 的骨架  
- 幫你設計一組實際可用的 prompts（繁中版）  
- 幫你把「只用本機 LLM + 本機向量庫」的版本拆出來（不依賴雲端）  

你比較想先動哪一塊？