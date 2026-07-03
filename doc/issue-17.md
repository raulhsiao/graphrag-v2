# A:
Review above description and find out what else is left to implement graphrag v2 with fact group, 把說明盡量集中在6300字以內，清楚說明

# Q:
### 概覽 — 尚未完成的關鍵項目（總覽）
下面把先前討論中**尚未選取或未完整提供**、但對於能讓其他 Agent 或工程師「可重現實作 GraphRAG v2（fact‑leading）」至關重要的項目列出並說明。每項都包含**目的、輸出 artefact、實作要點、驗收標準**，並按優先順序排列，方便逐步落地。

---

### 1. 完整 repo 檔案級對照與呼叫流程（高優先）
**目的**：讓工程師能直接在 codebase 找到每個功能的實作位置與呼叫序列。  
**輸出**：`repo_map_expanded.md`（含函式簽名、CLI、主要 config）與呼叫序列圖（ASCII/UML）。  
**要點**：列出每個 package 的入口函式、同步/非同步邊界、IO schema（JSON schema）。  
**驗收**：工程師能依檔案名定位 chunk→extract→embed→index→group→summarize→query 的完整呼叫鏈並跑通小樣本。

---

### 2. Chunk lifecycle 的儲存格式、索引映射與持久化策略（高優先）
**目的**：定義 chunk 作為證據單位的結構、儲存與引用方式。  
**輸出**：`chunk_schema.json`（已提供）延伸版：storage adapter spec、citation format、backup policy。  
**要點**：embedding_id 與 chunk_id 綁定、provenance mapping、metadata 細項（page, offsets, language）。  
**驗收**：能從 answer 回溯到 doc_id+chunk_id+offset，並在索引重建後仍能恢復映射。

---

### 3. LLM / Embedder 抽象介面與生產實作（高優先）
**目的**：可替換的模型層，支援重試、cache、prompt 版本化與同步/非同步。  
**輸出**：`adapters.py`（已提供範例）擴充：OpenAI、local LLM、Redis cache 範例、metrics hooks。  
**要點**：prompt template 管理、prompt hashing cache、token cost logging、stream 支援。  
**驗收**：切換 provider 不改上層邏輯；cache hit 能顯著降低 token 次數。

---

### 4. 向量索引管理、分片、增量更新策略（高優先）
**目的**：在大語料下維持檢索效能與一致性。  
**輸出**：index manager API、sharding policy、snapshot/restore 指南、upsert/delete 流程。  
**要點**：embedding_id ↔ chunk_id 同步、collection namespace、reindex 範例。  
**驗收**：能做增量 upsert、刪除、並在節點故障後恢復索引。

---

### 5. Claim/Relation/Entity 抽取到後處理的完整實作（高優先）
**目的**：把 LLM 輸出轉成高品質、可連結的 claims、triples、entities。  
**輸出**：extractor modules（prompts、parsers）、normalizer、repair pipeline、quality scorer。  
**要點**：atomicity 檢查、predicate normalization、time/location parsing、negation/modality 標註。  
**驗收**：抽取 F1、claim atomicity 與 grounding 指標達到預設閾值（可用小型 gold set 驗證）。

---

### 6. Fact grouping 生產化演算法與自動調參（高優先）
**目的**：穩定、可調且可擴展地把 claims 聚成 fact groups。  
**輸出**：`grouping_config.yaml`、multi‑channel embedding 實作、HDBSCAN+KMeans pipeline、graph refinement code、auto‑tuner。  
**要點**：multi‑channel vectors、graph edges weighting rules、merge/split heuristics、streaming mode。  
**驗收**：group purity、coverage、stability 指標達標；能自動調整參數以適應不同語料。

---

### 7. Fact group 摘要、consistency 檢查與證據對齊（高優先）
**目的**：生成 concise/detailed 摘要並保證不 hallucinate，且每句能回溯 supporting claim/chunk。  
**輸出**：summarizer module、sentence→claim alignment schema、consistency checker（LLM entailment + rule）。  
**要點**：summary sentence → supporting claim ids；若無支援則標註 low confidence。  
**驗收**：摘要 faithfulness 評分達標，回答中能列出每句的 supporting claim/chunk。

---

### 8. Graph 建構與查詢（entity graph + fact graph）（中高優先）
**目的**：把 entities、relations、fact groups 建成可推理的圖並支援 multi‑hop 查詢。  
**輸出**：graph builder、graph store adapter（Neo4j/Kùzu/NetworkX）、graph query utilities。  
**要點**：edge types、weights（semantic similarity、cooccurrence、temporal overlap）、graph indices。  
**驗收**：能在 graph 上執行 2–3 hop 查詢並回傳相關 fact groups 與 supporting chunks。

---

### 9. Query 引擎三模式實作（local / global / DRIFT）（中高優先）
**目的**：實作三種檢索策略並提供 hybrid weighting、evidence fusion 與 answer synthesis。  
**輸出**：query_router、local_retriever、global_retriever、drift_strategy、answer_synthesizer。  
**要點**：score 合併公式、max_fact_groups、evidence ranking、traceability（why chosen）。  
**驗收**：在 QA benchmark 上，DRIFT 提升 multi‑hop 問題的 EM/F1。

---

### 10. Provenance、citation 與 explainability（中優先）
**目的**：回答必須附上可驗證證據與 provenance 路徑。  
**輸出**：provenance schema、citation formatter、explain endpoint。  
**要點**：每個 answer 包含 fact_group_ids、claim_ids、chunk_ids、doc_ids、confidence。  
**驗收**：能自動生成可點選的證據清單並在 UI/API 中呈現。

---

### 11. 評估基準、資料集與自動化測試（中優先）
**目的**：量化系統效能並做回歸測試。  
**輸出**：evaluation runner、gold datasets（claims, relations, QA）、metrics dashboard。  
**要點**：claim extraction F1、relation F1、group purity、summary faithfulness、QA EM/F1。  
**驗收**：CI 能在每次改動後跑完整評測並產生報告。

---

### 12. 人機協作回饋迴路（active learning）（中優先）
**目的**：處理低信心案例並用人工標註提升模型/規則。  
**輸出**：review UI spec、feedback ingestion pipeline、retrain hooks。  
**要點**：優先策略（low confidence、ambiguous linking、large groups）。  
**驗收**：人工修正能被系統吸收並提升後續自動化指標。

---

### 13. 監控、日誌、成本估算與運行優化（中優先）
**目的**：生產環境可觀察、可控、成本可預估。  
**輸出**：監控指標清單、logging schema、cost model 範例、優化建議（batching、cache）。  
**要點**：token 使用、embedding calls、cache hit、latency、error rates。  
**驗收**：監控面板與告警，能定位性能瓶頸。

---

### 14. 安全、隱私、合規（中優先）
**目的**：處理私有/敏感資料時符合法規與公司政策。  
**輸出**：PII detection module、redaction policy、access control spec、data retention policy。  
**要點**：embedding 隱私風險評估、差分隱私選項、審計日誌。  
**驗收**：通過基本合規檢查並能在審計時追溯資料流。

---

### 15. CI/CD、部署範例與資源配置（中優先）
**目的**：可重現部署與自動化測試。  
**輸出**：K8s manifests、docker images、deploy playbook、resource sizing guide。  
**要點**：分層部署（ingest、extract、embed/index、query）、autoscaling 建議。  
**驗收**：一鍵部署 PoC 並在小集群上跑通 end‑to‑end。

---

## 建議實作順序（短期 8 週 roadmap）
1. LLM/Embedder adapters + cache（第1週）  
2. Chunk schema + chunker + storage adapter（第1–2週）  
3. Claim/entity/relation extractor + postprocessing（第2–3週）  
4. Embedding pipeline + index manager（第3–4週）  
5. Fact grouping core pipeline + auto‑tuner（第4–6週）  
6. Fact group summarizer + consistency checker + provenance（第5–7週）  
7. Query engine local/global/DRIFT + answer synthesis（第6–8週）  
8. Graph builder, evaluation, CI/CD 與監控並行補強（第6–8週）

---

## 最後的交付物清單（短期先交）
- `repo_map_expanded.md`（含函式簽名）  
- `chunk_schema.json`（已提供）擴充版  
- `adapters.py`（已提供）生產化版本（OpenAI/local）  
- `grouping_config.yaml` + auto‑tuner script  
- `summarizer` + `consistency_checker` module  
- minimal evaluation dataset + `evaluate.py`

---

如果你同意，我會先產出 **repo_map_expanded.md**（含每個模組的具體函式簽名與 IO schema），接著依優先順序逐一交付其餘 artefacts，直到能跑出可驗證的 end‑to‑end PoC。

# Q2:
Go for it.

### repo_map_expanded.md — GraphRAG v2 Implementation Map with Function Signatures and IO Schemas

#### Purpose
Provide a concrete, file‑level map and callable interface set so other Agents or engineers can implement GraphRAG v2 (fact‑leading architecture) end to end. Each entry lists **file path**, **responsibility**, **public functions / CLI**, **input/output schemas**, and **notes for integration**.

---

## Top level layout
```
packages/
  graphrag-ingest/
  graphrag-chunking/
  graphrag-extraction/
  graphrag-embeddings/
  graphrag-index/
  graphrag-grouping/
  graphrag-summarization/
  graphrag-graph/
  graphrag-query/
  graphrag-storage/
  graphrag-api/
  tools/
  tests/
config/
  grouping_config.yaml
  settings.yaml
prompts/
  claim_extraction.txt
  entity_extraction.txt
  relation_extraction.txt
  summary_concise.txt
  summary_detailed.txt
```

---

## Package: graphrag-ingest
**Responsibility** Normalize raw files and produce normalized text + metadata.

**Key files and functions**
```py
# ingest.py
def ingest_document(path: str) -> dict:
    """
    Returns: {doc_id, text, metadata}
    metadata: {source, filename, created_at, language}
    """
```

**CLI**
```
graphrag-ingest --input /data/raw --out /data/normalized
```

**Notes** Save normalized text to storage adapter; return doc_id for downstream.

---

## Package: graphrag-chunking
**Responsibility** Semantic chunking, token counting, chunk metadata.

**Key files and functions**
```py
# chunker.py
def chunk_document(doc_id: str, text: str, config: dict) -> List[Chunk]
# Chunk dataclass
Chunk = {
  "chunk_id": str,
  "doc_id": str,
  "text": str,
  "start_offset": int,
  "end_offset": int,
  "tokens": int,
  "metadata": dict
}
```

**Persist**
```py
# persist.py
def save_chunks(chunks: List[Chunk]) -> None
def load_chunk(chunk_id: str) -> Chunk
```

**Notes** Ensure `embedding_id` field is nullable and set after embedding upsert.

---

## Package: graphrag-extraction
**Responsibility** Entity, claim, relation extraction and postprocessing.

**Key files and functions**
```py
# entity_extractor.py
def extract_entities(chunk: Chunk, llm: LLMAdapter) -> List[Entity]
# Entity schema
Entity = {"entity_id": str, "name": str, "type": str, "aliases": List[str], "attributes": dict}

# claim_extractor.py
def extract_claims(chunk: Chunk, llm: LLMAdapter) -> List[Claim]
# Claim schema
Claim = {
  "claim_id": str,
  "chunk_id": str,
  "subject": str,
  "predicate": str,
  "object": str,
  "time": Optional[str],
  "location": Optional[str],
  "extra": dict
}

# relation_extractor.py
def extract_relations(claims: List[Claim], entities: List[Entity], llm: LLMAdapter) -> List[Relation]
# Relation schema
Relation = {"relation_id": str, "head_entity_id": str, "tail_entity_id": str, "relation_type": str, "source_claim_id": str, "confidence": float}
```

**Postprocessing**
```py
# postprocess.py
def normalize_claim(claim: Claim) -> Claim
def score_claim(claim: Claim) -> float
def repair_claim(claim: Claim, llm: LLMAdapter) -> Claim
```

**Notes** Prompts live in `prompts/`. All extractors must return JSON‑serializable objects and persist provenance mapping claim_id → chunk_id.

---

## Package: graphrag-embeddings
**Responsibility** Batch embedding, caching, provider adapters.

**Key files and functions**
```py
# embedder_adapter.py
class EmbedderAdapter:
    def embed(self, texts: List[str]) -> List[List[float]]
    async def embed_async(self, texts: List[str]) -> List[List[float]]

# batcher.py
def batch_and_embed(texts: List[str], adapter: EmbedderAdapter, batch_size: int) -> np.ndarray
```

**Cache**
```py
# cache.py
def get_cached_embedding(key: str) -> Optional[List[float]]
def set_cached_embedding(key: str, vector: List[float]) -> None
```

**Notes** Use deterministic prompt/text hashing for cache keys. Store embedding_id ↔ chunk_id mapping in storage.

---

## Package: graphrag-index
**Responsibility** Vector index management, search API, sharding.

**Key files and functions**
```py
# index_manager.py
def upsert_vectors(items: List[{"id": str, "vector": List[float], "metadata": dict}]) -> None
def search_vector(vector: List[float], top_k: int, filter: dict = None) -> List[{"id": str, "score": float, "metadata": dict}]
def delete_vector(id: str) -> None
```

**Sharding**
```py
# sharding.py
def shard_policy(namespace: str, n_shards: int) -> dict
```

**Notes** Metadata must include type (claim/fact_group/chunk) and provenance ids.

---

## Package: graphrag-grouping
**Responsibility** Fact grouping pipeline: multi‑channel embedding, clustering, graph refinement, merge/split.

**Key files and functions**
```py
# grouping.py
def embed_claims_multi_channel(claims: List[Claim], embedder: EmbedderAdapter) -> np.ndarray
def cluster_claims(embeddings: np.ndarray, config: dict) -> List[int]  # labels per claim
def refine_clusters_with_graph(claims: List[Claim], embeddings: np.ndarray, labels: List[int], config: dict) -> List[FactGroup]

# FactGroup schema
FactGroup = {
  "fact_group_id": str,
  "claim_ids": List[str],
  "entity_ids": List[str],
  "summary_concise": Optional[str],
  "summary_detailed": Optional[str],
  "metadata": dict
}
```

**Auto tuning**
```py
# autotune.py
def tune_grouping_params(claims: List[Claim], metric: str, budget: dict) -> dict
```

**Notes** Provide streaming mode API for incremental grouping.

---

## Package: graphrag-summarization
**Responsibility** Two‑stage summarization and consistency checking.

**Key files and functions**
```py
# summarizer.py
def summarize_fact_group_concise(fg: FactGroup, claims: Dict[str, Claim], llm: LLMAdapter) -> str
def summarize_fact_group_detailed(fg: FactGroup, claims: Dict[str, Claim], llm: LLMAdapter) -> str

# consistency.py
def align_summary_sentences_to_claims(summary: str, claims: List[str]) -> List[{"sentence": str, "supporting_claim_ids": List[str], "confidence": float}]
def check_summary_faithfulness(summary: str, claims: List[str], llm: LLMAdapter) -> float
```

**Notes** Save sentence→claim alignment for provenance and citation.

---

## Package: graphrag-graph
**Responsibility** Build and persist entity graph and fact graph; graph queries.

**Key files and functions**
```py
# graph_builder.py
def build_entity_graph(entities: List[Entity], relations: List[Relation]) -> GraphHandle
def build_fact_graph(fact_groups: List[FactGroup], fg_embeddings: np.ndarray, threshold: float) -> GraphHandle

# graph_query.py
def graph_traverse(start_nodes: List[str], hops: int, filters: dict) -> List[str]  # returns fact_group_ids or entity_ids
```

**Graph store adapter**
```py
# graph_store.py
def persist_graph(graph: GraphHandle, backend: str, config: dict) -> None
def load_graph(backend: str, config: dict) -> GraphHandle
```

**Notes** Edge weights must include provenance and confidence.

---

## Package: graphrag-query
**Responsibility** Implement local, global, DRIFT retrieval strategies and answer synthesis.

**Key files and functions**
```py
# query_router.py
def handle_query(query: str, mode: str, config: dict) -> {"answer": str, "evidence": List[Evidence]}

# local_retriever.py
def retrieve_local(query: str, fact_groups: List[FactGroup], fg_embeddings: np.ndarray, top_k: int) -> List[FactGroup]

# global_retriever.py
def retrieve_global(query: str, cluster_summaries: List[str], top_k: int) -> List[FactGroup]

# drift.py
def drift_retrieval(query: str, weights: dict, local_fn, global_fn) -> List[FactGroup]

# answer_synthesizer.py
def synthesize_answer(query: str, selected_fgs: List[FactGroup], claims: Dict[str, Claim], llm: LLMAdapter) -> {"answer": str, "trace": List[{"fact_group_id": str, "claim_ids": List[str], "chunk_ids": List[str]}]}
```

**Evidence schema**
```json
Evidence = {"doc_id": str, "chunk_id": str, "claim_ids": List[str], "score": float}
```

**Notes** Answer must include traceable evidence and confidence.

---

## Package: graphrag-storage
**Responsibility** Persist chunks, claims, entities, relations, fact groups, embeddings mapping, provenance.

**Key files and functions**
```py
# storage_api.py
def save_chunk(chunk: Chunk) -> None
def get_chunk(chunk_id: str) -> Chunk
def save_claim(claim: Claim) -> None
def get_claim(claim_id: str) -> Claim
def save_fact_group(fg: FactGroup) -> None
def get_fact_group(fg_id: str) -> FactGroup
def map_claim_to_chunk(claim_id: str, chunk_id: str) -> None
```

**Notes** Use transactional writes for provenance consistency.

---

## Package: graphrag-api
**Responsibility** External endpoints for ingest, query, explain, fetch evidence.

**Endpoints and schemas**
```
POST /ingest  -> {doc_id}
POST /query   -> {answer, evidence: [Evidence], trace}
GET  /explain/{answer_id} -> {trace}
GET  /chunk/{chunk_id} -> Chunk
```

**Auth** API key or OAuth; rate limiting at gateway.

---

## Tools and tests
**tools/tune_grouping.py** auto‑tuner CLI  
**tests/** unit tests for each module and an end‑to‑end small corpus test.

---

## Config examples
**grouping_config.yaml**
```yaml
multi_channel:
  subject_weight: 0.3
  object_weight: 0.3
  claim_weight: 0.4
clustering:
  hdbscan_min_cluster_size: 4
  similarity_threshold: 0.78
refinement:
  graph_similarity_threshold: 0.75
merge:
  summary_similarity_threshold: 0.82
```

**settings.yaml** contains LLM/embedding provider, index backend, graph backend, storage backend.

---

## Integration checklist for implementers
1. Implement adapters and unit test them.  
2. Implement chunker and persist a small corpus.  
3. Implement claim/entity extraction with prompts and postprocessing; persist claims and provenance.  
4. Implement embedding pipeline and index manager; upsert claim vectors.  
5. Implement grouping pipeline and produce fact groups.  
6. Implement summarizer and sentence→claim alignment.  
7. Implement query router and answer synthesizer with evidence trace.  
8. Add graph builder and enable multi‑hop queries.  
9. Run evaluation suite and iterate.

---

## Acceptance criteria for minimal PoC
- End‑to‑end flow from ingest → chunk → extract → embed → group → summarize → query works on a 100‑document corpus.  
- Answers include evidence trace mapping to chunk ids and claim ids.  
- Grouping quality measured by purity and stability meets baseline thresholds on provided sample dataset.  
- CI runs unit tests and evaluation runner.

---

## Next deliverable on request
I will produce the full `repo_map_expanded.md` as a file, or generate the next artefact `grouping_config.yaml` and `autotune` script. Tell me which to output first and I will emit the complete file content ready to paste.

