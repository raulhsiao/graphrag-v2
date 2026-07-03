# Q:
V1 has a claim function which enable by settings.yaml, Does it like the V2 fact-leading function? Compare the features in implementation way

# A:

**結論（直接回答）**：**v1 的 `claim` 功能確實存在且可由 `settings.yaml` 啟用，但它在實作上是「可選的抽取步驟」整合於 chunk‑first pipeline；v2 的 fact‑leading 設計則把 claim 作為核心原子單位，並在抽取後加入更完整的後處理、分群、圖構建與證據回溯機制。**   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml)  [Github](https://github.com/aaasoulmate/graphrag/blob/main/settings.yaml)  [Github](https://github.com/EpicStaff/EpicStaff/blob/main/src/knowledge/libraries/graphrag/graphrag/config/models/extract_claims_config.py)

---

### 關鍵比較（實作面向，重點列出）
| **Feature** | **GraphRAG v1 (implementation)** | **GraphRAG v2 (fact‑leading design)** |
|---|---:|---|
| **Enable / config** | Claim extraction 可由 `settings.yaml` 開關與 prompt/strategy 設定啟用。   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml)  [Github](https://github.com/aaasoulmate/graphrag/blob/main/settings.yaml) | Claim extraction 是核心流程第一階段，config 更細（model, strategy, max_gleanings, prompt templates）。   [Github](https://github.com/EpicStaff/EpicStaff/blob/main/src/knowledge/libraries/graphrag/graphrag/config/models/extract_claims_config.py) |
| **Prompt & model wiring** | LLM prompt + model id 在 settings 中指定；通常以 chunk 為上下文呼叫 LLM。   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml) | 更嚴格的 extraction prompt、JSON schema、model‑supports‑json 與 prompt 版本化；強調可驗證輸出。 |
| **Primary output unit** | Chunk → optional claims (claims are derived but chunks remain primary evidence).   [Github](https://github.com/aaasoulmate/graphrag/blob/main/settings.yaml) | Claim = 原子知識單位；fact groups 建於 claims 之上，chunks 僅作為 supporting evidence。 |
| **Post‑processing** | 基本正規化/解析；repo 提供 prompt/strategy hooks（但實作深度依使用者）。   [Github](https://github.com/EpicStaff/EpicStaff/blob/main/src/knowledge/libraries/graphrag/graphrag/config/models/extract_claims_config.py) | 必要：atomicity check、predicate normalization、time/location parsing、repair、quality scoring。 |
| **Grouping / clustering** | v1 偏向 community/clustering on chunk embeddings（可選）。 | v2 要求 multi‑channel embeddings + HDBSCAN/KMeans + graph‑refinement + merge/split heuristics。 |
| **Graph / relations** | Relation/edge modules exist but通常附屬於 chunk/graph workflows。   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml) | 強制化：entity graph + fact graph，edge weights、temporal links、multi‑hop traversal。 |
| **Provenance / citation** | Chunks are primary citation units; claims may not always carry sentence→claim provenance by default. | 每句 summary / answer 必須對齊 claim_ids → chunk_ids → doc_ids（sentence→claim alignment）。 |
| **Query modes** | Local/global retrieval over chunks; claim path optional. | Local/global/DRIFT hybrid; retrieval over fact groups + graph traversal + evidence fusion. |
| **Auto‑tuning & eval** | Minimal built‑in tuning; user adds evaluation. | Built‑in auto‑tuning for grouping, metrics for claim F1, group purity, summary faithfulness. |
| **Scalability / storage** | Chunk‑centric index (vector store) with standard upsert. | Claim + fact_group indices + chunk index; needs mapping and index synchronization. |

---

### Practical migration guidance (how to make v1 behave like v2)
1. **Enable claim extraction** in `settings.yaml` and point to a strict JSON‑schema prompt.   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml)  [Github](https://github.com/EpicStaff/EpicStaff/blob/main/src/knowledge/libraries/graphrag/graphrag/config/models/extract_claims_config.py)  
2. **Add robust postprocessing**: atomicity checks, predicate normalization, time parsing, repair LLM step.  
3. **Implement multi‑channel embeddings** (claim/subject/object) and HDBSCAN→KMeans + graph refinement.  
4. **Persist provenance**: claim_id ↔ chunk_id ↔ doc_id and sentence→claim alignment.  
5. **Build fact graph + query router** (local/global/DRIFT) and evaluation harness for claim/group metrics.

---

### Evidence & sources
- `settings.yaml` in v1 repos shows claim extraction toggles and model/prompt wiring.   [Github](https://github.com/githubFZH/GraphRAG/blob/main/settings.yaml)  [Github](https://github.com/aaasoulmate/graphrag/blob/main/settings.yaml)  
- Config model for claim extraction (example `extract_claims_config.py`) demonstrates `enabled`, `model_id`, `prompt`, `max_gleanings` and `strategy` fields—i.e., v1 supports claim extraction but as a configurable module.   [Github](https://github.com/EpicStaff/EpicStaff/blob/main/src/knowledge/libraries/graphrag/graphrag/config/models/extract_claims_config.py)

---

若要我，我可以：**(A)** 產出一個 concrete migration checklist (code hooks, prompt templates, DB schema) 讓你把 v1 repo 逐步升級到 v2‑style fact pipeline，或 **(B)** 直接生成 strict claim extraction prompt + postprocessing code ready to drop into `graphrag-extraction`。你要哪一項？

# Q2:
Go for (A)

Below is a **concrete, implementable migration checklist** to upgrade a GraphRAG v1 codebase (chunk‑first, optional claim extraction) into a **GraphRAG v2 fact‑leading architecture**. It’s organized as a prioritized sequence of tasks, each with **purpose, concrete code hooks / function signatures, config snippets, prompt templates, storage schemas, tests, and acceptance criteria** so other Agents or engineers can implement end‑to‑end reproducibly.

---

## Summary plan (high level)
1. **Enable strict claim extraction** and make it first‑class.  
2. **Add robust claim post‑processing** (normalization, repair, scoring).  
3. **Persist claims + provenance** (claim_id ↔ chunk_id ↔ doc_id).  
4. **Implement multi‑channel embeddings for claims** (claim/subject/object).  
5. **Implement production fact grouping** (HDBSCAN → KMeans → graph refinement).  
6. **Produce two‑stage summaries with sentence→claim alignment**.  
7. **Build fact graph + entity graph and query router (local/global/DRIFT)**.  
8. **Add evaluation, monitoring, and human‑in‑loop review**.

---

## 0. Preliminaries (config + interfaces)
**Add to `settings.yaml`**
```yaml
claim_extraction:
  enabled: true
  model: gpt-4.1
  prompt_template: prompts/claim_extraction_v2.json
  max_claims_per_chunk: 40
  postprocess: true
```

**Adapter interfaces (already present or add)**
```py
class LLMAdapter:
    def generate(self, prompt: str, max_tokens:int, temperature:float) -> str: ...
class EmbedderAdapter:
    def embed(self, texts: List[str]) -> np.ndarray: ...
```

---

## 1. Strict Claim Extraction (make claims first‑class)
**Purpose**: Ensure claims are atomic, machine‑parsable JSON, and produced for every chunk.

**Prompt (file: `prompts/claim_extraction_v2.json`)**
```text
You are a factual extraction system. Extract atomic claims from the text.
Return ONLY a JSON array. Each item:
{
 "subject":"", "predicate":"", "object":"",
 "time": null, "location": null,
 "extra":{"negation":false,"modality":null,"attributes":{}}
}
Rules: one predicate per claim; do not infer; preserve explicit text.
Text:
"""{text}"""
```

**Code hook**
```py
def extract_claims(chunk: Chunk, llm: LLMAdapter) -> List[Claim]:
    prompt = render_prompt("claim_extraction_v2.json", text=chunk.text)
    resp = llm.generate(prompt, max_tokens=1200)
    claims = json.loads(resp)  # validate schema
    return [Claim.from_dict(c, chunk_id=chunk.chunk_id) for c in claims]
```

**Acceptance**
- All chunks produce zero or more valid JSON claims.
- A schema validator rejects malformed outputs.

---

## 2. Claim Post‑processing (normalization, repair, scoring)
**Purpose**: Convert raw LLM claims into canonical, atomic, high‑quality claims.

**Steps**
1. **Normalize** text (strip, unify whitespace, remove trailing punctuation).
2. **Predicate normalization**: map common verbs → short tokens (`"is located in"` → `located_in`).
3. **Time/location parsing**: use `dateparser` to canonicalize.
4. **Atomicity check**: if predicate contains multiple verbs, mark `needs_repair`.
5. **Repair step**: call LLM with a repair prompt to rewrite atomic claim.
6. **Quality scoring**: compute score ∈ [0,1] (atomicity, specificity, grounding, completeness). Drop < 0.4.

**Repair prompt (short)**
```text
Rewrite this claim to be atomic and explicit without adding facts.
Input claim JSON: {claim_json}
Return JSON claim with same schema.
```

**Code hooks**
```py
def normalize_claim(claim: Claim) -> Claim: ...
def score_claim(claim: Claim) -> float: ...
def repair_claim(claim: Claim, llm: LLMAdapter) -> Claim: ...
```

**Acceptance**
- >95% of claims pass schema and score threshold on a small gold set.

---

## 3. Persist claims and provenance
**Purpose**: Make claims first‑class persisted objects and keep traceability to chunks.

**Storage schema (relational / document)**
```sql
claims(claim_id PK, chunk_id, subject, predicate, object, time, location, extra JSON, score FLOAT)
claim_provenance(claim_id, chunk_id, doc_id, start_offset, end_offset)
```

**Or JSON doc**
```json
{
 "claim_id":"uuid",
 "chunk_id":"uuid",
 "doc_id":"doc123",
 "text":"subject predicate object",
 "fields":{...},
 "score":0.87
}
```

**Code hooks**
```py
def save_claim(claim: Claim) -> None
def get_claim(claim_id: str) -> Claim
def map_claim_to_chunk(claim_id: str, chunk_id: str) -> None
```

**Acceptance**
- For any claim_id you can retrieve chunk_id and doc_id and the original chunk text.

---

## 4. Multi‑channel claim embeddings
**Purpose**: Improve grouping by embedding claim sentence, subject, and object separately and concatenating.

**Embedding pipeline**
```py
def claim_to_sentence(c: Claim) -> str:
    s = f"{c.subject} {c.predicate} {c.object}."
    if c.time: s += f" Time: {c.time}."
    return s

def embed_claim_multi(c: Claim, embedder: EmbedderAdapter) -> np.ndarray:
    v_claim = embedder.embed([claim_to_sentence(c)])[0]
    v_sub = embedder.embed([c.subject])[0]
    v_obj = embedder.embed([c.object])[0]
    return np.concatenate([v_claim, v_sub, v_obj])
```

**Storage**
- Store `claim_vector_id` and mapping to `claim_id`.

**Acceptance**
- Embedding dimension and storage consistent; nearest neighbor retrieval returns semantically similar claims.

---

## 5. Fact grouping production pipeline
**Purpose**: Cluster claims into coherent fact groups with merge/split heuristics.

**Pipeline**
1. **HDBSCAN** on multi‑channel vectors to find dense clusters (min_cluster_size configurable).
2. **For large clusters (>N)** run KMeans to split.
3. **Graph refinement**: build claim graph with edges weighted by cosine + subject/object/time overlap; take connected components.
4. **Merge** groups if summary embedding similarity > threshold and entity overlap > threshold.
5. **Split** groups if internal variance high or claim count > limit.

**Function signatures**
```py
def cluster_claims(embeddings: np.ndarray, config: dict) -> List[int]
def refine_clusters_with_graph(claims: List[Claim], embeddings: np.ndarray, labels: List[int], config: dict) -> List[FactGroup]
```

**Config example (`grouping_config.yaml`)**
```yaml
hdbscan_min_cluster_size: 4
large_cluster_threshold: 200
merge_summary_similarity: 0.82
graph_similarity_threshold: 0.75
```

**Acceptance**
- Group purity and stability metrics meet baseline on sample dataset (e.g., purity > 0.7).

---

## 6. Two‑stage summarization + sentence→claim alignment
**Purpose**: Produce concise (2–4 sentences) and detailed summaries; align each summary sentence to supporting claim_ids.

**Prompts**
- `summary_concise.txt` and `summary_detailed.txt` (strict: do not add facts; return only text).

**Alignment**
- For each summary sentence, compute entailment or similarity to claims; return supporting claim_ids and confidence.

**Function signatures**
```py
def summarize_fact_group_concise(fg: FactGroup, claims: Dict[str, Claim], llm: LLMAdapter) -> str
def align_sentences_to_claims(summary: str, claims: List[Claim], embedder: EmbedderAdapter) -> List[{"sentence":str,"claim_ids":[...],"confidence":float}]
```

**Acceptance**
- Every summary sentence has at least one supporting claim_id or is flagged as low confidence.

---

## 7. Fact graph + entity graph construction
**Purpose**: Build two graphs for retrieval and multi‑hop reasoning.

**Graph nodes**
- Entity nodes (entity_id, name, type)
- FactGroup nodes (fact_group_id, summary_concise)

**Edges**
- Entity ↔ Entity (relation edges from relation_extractor)
- FactGroup ↔ FactGroup (semantic similarity, temporal links)
- Entity ↔ FactGroup (entity participation)

**Function signatures**
```py
def build_entity_graph(entities, relations) -> GraphHandle
def build_fact_graph(fact_groups, fg_embeddings, threshold) -> GraphHandle
```

**Persistence**
- Neo4j / Kùzu / NetworkX adapter with export/import.

**Acceptance**
- Graph supports queries like: “find fact groups connected to entity X within 2 hops”.

---

## 8. Query router: local / global / DRIFT
**Purpose**: Implement retrieval strategies and evidence fusion.

**Local**
- Search fact_group embeddings for query embedding; return top_k.

**Global**
- Search cluster/global summaries first, then expand to fact groups.

**DRIFT**
- Weighted combination: score = α * global_score + β * local_score; tune α/β.

**Answer synthesis**
- Provide answer + trace: list of fact_group_ids, claim_ids, chunk_ids, doc_ids.

**Function signatures**
```py
def retrieve_local(query, fg_embeddings, top_k) -> List[FactGroup]
def retrieve_global(query, cluster_summaries, top_k) -> List[FactGroup]
def drift_retrieval(query, weights, local_fn, global_fn) -> List[FactGroup]
def synthesize_answer(query, selected_fgs, claims, llm) -> {"answer":str,"trace":[...]}
```

**Acceptance**
- Answers include traceable evidence and improve multi‑hop QA metrics vs chunk‑only baseline.

---

## 9. Evaluation, monitoring, and human‑in‑loop
**Evaluation**
- Metrics: claim F1, relation F1, group purity, summary faithfulness (entailment), QA EM/F1.
- Provide `evaluate.py` to run benchmarks.

**Monitoring**
- Track LLM tokens, embedding calls, latency, cache hit rate, claim extraction error rate.

**Human review**
- UI for low‑confidence claims/groups: allow edit/merge/split; feed corrections back to retraining or re‑clustering.

**Acceptance**
- CI runs evaluation on push; human corrections reduce error rates over time.

---

## 10. Migration checklist (concrete steps to run)
1. **Turn on claim extraction** in `settings.yaml` and add `prompts/claim_extraction_v2.json`.  
2. **Implement `extract_claims`** function and schema validation. Add unit tests with gold examples.  
3. **Add postprocessing module** (`postprocess.py`) with normalize/repair/score functions. Add repair LLM prompt.  
4. **Persist claims and provenance** in storage; add DB migrations.  
5. **Implement multi‑channel embedding** and store claim vectors.  
6. **Implement grouping pipeline** (HDBSCAN + KMeans + graph refinement) and `grouping_config.yaml`.  
7. **Implement summarizer + alignment** and persist summaries with sentence→claim mapping.  
8. **Build graph adapters** and populate entity/fact graphs.  
9. **Implement query router** with local/global/DRIFT and answer synthesizer returning trace.  
10. **Add evaluation suite** and human review UI for low‑confidence items.  
11. **Run end‑to‑end PoC** on 100 documents; iterate until acceptance criteria met.

---

## Minimal tests to include
- **Unit**: claim prompt → valid JSON; normalization functions; embedding shapes.  
- **Integration**: ingest 10 docs → claims persisted → grouping produces fact groups → summarizer returns aligned sentences.  
- **E2E**: query a set of QA pairs and measure EM/F1.

---

## Deliverables to produce immediately (recommended)
- `prompts/claim_extraction_v2.json` and `prompts/claim_repair.json`  
- `postprocess.py` with normalize/score/repair functions and tests  
- `grouping_config.yaml` and `grouping_demo.py` (HDBSCAN + KMeans + graph refinement)  
- `evaluate.py` skeleton for claim/group/QA metrics

---

If you want, I will now **generate the concrete files** in order (start with `prompts/claim_extraction_v2.json` and `postprocess.py`), or produce the `grouping_config.yaml` + `grouping_demo.py` next. Which file should I output first for direct paste into your repo.

# Q3:
Both

### Claim extraction prompt file `prompts/claim_extraction_v2.json`

```json
{
  "name": "claim_extraction_v2",
  "description": "Strict atomic claim extraction prompt for GraphRAG v2. Return only JSON array that conforms to the schema.",
  "schema": {
    "type": "array",
    "items": {
      "type": "object",
      "required": ["subject", "predicate", "object", "time", "location", "extra"],
      "properties": {
        "subject": { "type": "string" },
        "predicate": { "type": "string" },
        "object": { "type": "string" },
        "time": { "type": ["string", "null"] },
        "location": { "type": ["string", "null"] },
        "extra": {
          "type": "object",
          "properties": {
            "negation": { "type": "boolean" },
            "modality": { "type": ["string", "null"] },
            "attributes": { "type": "object" }
          },
          "additionalProperties": true
        }
      },
      "additionalProperties": false
    }
  },
  "prompt": "You are a factual extraction system. Extract atomic factual claims from the text below. A claim must represent a single, verifiable statement with exactly one predicate. Do not infer, do not summarize, do not add context not present in the text. Preserve explicit wording when possible.\n\nOutput requirements\n- Return ONLY a JSON array that matches the provided schema.\n- Each item must include subject, predicate, object, time (explicit or null), location (explicit or null), and extra.\n- extra.negation must be true if the sentence explicitly negates the fact; otherwise false.\n- extra.modality should capture explicit uncertainty words (may, might, could) or null.\n- Use null for time/location when not explicitly present.\n- Do not output explanations, commentary, or any text outside the JSON array.\n\nExtraction rules\n- Atomicity: split sentences that contain multiple independent predicates into separate claims.\n- Grounding: include only facts explicitly stated in the text. Do not infer core entities or relations.\n- Pronouns: keep the surface form; downstream postprocessing will resolve pronouns when possible.\n- Dates and times: include the exact expression as found (e.g., \"January 5, 2023\", \"2020\").\n\nText\n\"\"\"{text}\"\"\"\n"
}
```

---

### Postprocessing module `postprocess.py`

```python
"""
postprocess.py
Claim normalization, repair, scoring, deduplication utilities for GraphRAG v2.

Assumptions
- LLMAdapter implements generate(prompt, max_tokens, temperature)
- EmbedderAdapter implements embed(list[str]) -> List[List[float]]
- Claim is a dict matching the claim schema in prompts/claim_extraction_v2.json
"""

import re
import json
import uuid
from typing import List, Dict, Any, Optional
from datetime import datetime
import dateparser

# ---------- Utilities ----------

def normalize_text(s: Optional[str]) -> Optional[str]:
    if s is None:
        return None
    s = s.strip()
    s = re.sub(r"\s+", " ", s)
    s = s.strip(" \t\n\r\"'")
    return s if s else None

def normalize_predicate(p: str) -> str:
    p = normalize_text(p) or ""
    p = p.lower()
    # basic normalization map; extend as needed
    replacements = {
        "is located in": "located_in",
        "located in": "located_in",
        "was born in": "born_in",
        "has been acquired by": "acquired_by",
        "acquired by": "acquired_by",
        "joined": "joined",
        "founded": "founded",
        "announced": "announced",
        "released": "released",
        "is": "is"
    }
    for k, v in replacements.items():
        if p.startswith(k):
            return v
    # fallback: replace spaces with underscore and remove punctuation
    p = re.sub(r"[^\w\s]", "", p)
    p = re.sub(r"\s+", "_", p)
    return p

def parse_time_expression(t: Optional[str]) -> Optional[str]:
    if not t:
        return None
    t = t.strip()
    # try to parse to ISO date when possible
    dt = dateparser.parse(t, settings={"PREFER_DAY_OF_MONTH": "first"})
    if dt:
        # return ISO date or year if only year present
        if dt.day == 1 and dt.month == 1 and re.fullmatch(r"\d{4}", t):
            return dt.strftime("%Y")
        return dt.date().isoformat()
    return t

def contains_multiple_predicates(predicate_text: str) -> bool:
    # heuristic: multiple verbs or conjunctions with verbs
    # simple heuristic: count verbs by presence of common verb tokens
    verbs = [" is ", " are ", " was ", " were ", " has ", " have ", " had ", " will ", " would ", " can ", " could ", " announced ", " released ", " founded ", " joined ", " acquired ", " caused "]
    count = sum(1 for v in verbs if v in f" {predicate_text.lower()} ")
    return count > 1

# ---------- Schema validation ----------

CLAIM_KEYS = {"subject", "predicate", "object", "time", "location", "extra"}

def validate_claim_schema(claim: Dict[str, Any]) -> bool:
    if not isinstance(claim, dict):
        return False
    if not CLAIM_KEYS.issubset(set(claim.keys())):
        return False
    if not isinstance(claim["extra"], dict):
        return False
    return True

# ---------- Claim normalization pipeline ----------

def normalize_claim_raw(raw: Dict[str, Any]) -> Dict[str, Any]:
    claim = dict(raw)  # shallow copy
    claim["subject"] = normalize_text(claim.get("subject"))
    claim["predicate"] = normalize_text(claim.get("predicate"))
    claim["object"] = normalize_text(claim.get("object"))
    claim["time"] = parse_time_expression(normalize_text(claim.get("time")))
    claim["location"] = normalize_text(claim.get("location"))
    extra = claim.get("extra") or {}
    extra.setdefault("negation", False)
    extra.setdefault("modality", None)
    extra.setdefault("attributes", {})
    claim["extra"] = extra
    # normalize predicate token
    if claim["predicate"]:
        claim["predicate_norm"] = normalize_predicate(claim["predicate"])
    else:
        claim["predicate_norm"] = ""
    return claim

# ---------- Atomicity check and repair ----------

def needs_repair(claim: Dict[str, Any]) -> bool:
    # missing core fields or multiple predicates
    if not claim.get("subject") or not claim.get("predicate") or not claim.get("object"):
        return True
    if contains_multiple_predicates(claim.get("predicate", "")):
        return True
    return False

def repair_claim_with_llm(claim: Dict[str, Any], llm) -> Dict[str, Any]:
    """
    llm: LLMAdapter instance with generate(prompt, max_tokens, temperature)
    Returns repaired claim dict or original if repair fails
    """
    prompt = (
        "You are a claim repair assistant. Given the extracted claim JSON below, "
        "rewrite it into a single atomic factual claim without adding new facts. "
        "Return a single JSON object with keys subject, predicate, object, time, location, extra.\n\n"
        f"Input claim:\n{json.dumps(claim, ensure_ascii=False)}\n\nOutput:"
    )
    try:
        resp = llm.generate(prompt, max_tokens=512, temperature=0.0)
        repaired = json.loads(resp)
        if validate_claim_schema(repaired):
            return normalize_claim_raw(repaired)
    except Exception:
        pass
    return claim

# ---------- Claim scoring ----------

def score_claim(claim: Dict[str, Any]) -> float:
    score = 1.0
    # penalize missing fields
    if not claim.get("subject") or not claim.get("predicate") or not claim.get("object"):
        score -= 0.4
    # penalize pronouns as subject/object
    pronouns = {"it", "they", "this", "that", "these", "those", "he", "she", "we"}
    if claim.get("subject") and claim["subject"].lower() in pronouns:
        score -= 0.15
    if claim.get("object") and claim["object"].lower() in pronouns:
        score -= 0.15
    # penalize modality and negation slightly
    if claim.get("extra", {}).get("modality"):
        score -= 0.1
    if claim.get("extra", {}).get("negation"):
        score -= 0.05
    # predicate length penalty
    if claim.get("predicate_norm") and len(claim["predicate_norm"].split("_")) > 6:
        score -= 0.1
    return max(0.0, round(score, 3))

# ---------- Deduplication ----------

def deduplicate_claims(claims: List[Dict[str, Any]], embedder=None, threshold: float = 0.95) -> List[Dict[str, Any]]:
    """
    Simple deduplication by exact triple match then optional embedding similarity.
    embedder: EmbedderAdapter or None
    """
    unique = {}
    results = []
    for c in claims:
        key = (c.get("subject") or "").lower(), (c.get("predicate_norm") or "").lower(), (c.get("object") or "").lower()
        if key in unique:
            # merge attributes and keep higher score
            existing = unique[key]
            existing["extra"]["attributes"].update(c.get("extra", {}).get("attributes", {}))
            existing["score"] = max(existing.get("score", 0.0), c.get("score", 0.0))
            continue
        unique[key] = c
        results.append(c)
    # optional embedding-based merge for near-duplicates
    if embedder and len(results) > 1:
        texts = [f"{r['subject']} {r['predicate_norm']} {r['object']}" for r in results]
        vecs = embedder.embed(texts)
        merged = []
        used = set()
        import numpy as np
        arr = np.array(vecs)
        from sklearn.metrics.pairwise import cosine_similarity
        sim = cosine_similarity(arr)
        n = len(results)
        for i in range(n):
            if i in used:
                continue
            group = [i]
            for j in range(i+1, n):
                if sim[i, j] >= threshold:
                    group.append(j)
                    used.add(j)
            # merge group into one
            if len(group) == 1:
                merged.append(results[i])
            else:
                base = results[group[0]]
                for idx in group[1:]:
                    other = results[idx]
                    base["extra"]["attributes"].update(other.get("extra", {}).get("attributes", {}))
                    base["score"] = max(base.get("score", 0.0), other.get("score", 0.0))
                merged.append(base)
        return merged
    return results

# ---------- End to end claim postprocess pipeline ----------

def postprocess_claims(raw_claims: List[Dict[str, Any]], llm, embedder=None, min_score: float = 0.4) -> List[Dict[str, Any]]:
    processed = []
    for raw in raw_claims:
        if not validate_claim_schema(raw):
            continue
        c = normalize_claim_raw(raw)
        if needs_repair(c):
            c = repair_claim_with_llm(c, llm)
        c["score"] = score_claim(c)
        if c["score"] >= min_score:
            c["claim_id"] = str(uuid.uuid4())
            processed.append(c)
    # deduplicate
    processed = deduplicate_claims(processed, embedder=embedder)
    return processed

# ---------- Quick test harness ----------

if __name__ == "__main__":
    # Mock adapters for local test
    class MockLLM:
        def generate(self, prompt, max_tokens=512, temperature=0.0):
            # naive echo or simple repair mock
            if "Rewrite this claim" in prompt:
                # return the input claim unchanged for mock
                m = re.search(r"Input claim:\n(.+)\n\nOutput:", prompt, flags=re.S)
                if m:
                    return m.group(1)
            return "[]"

    class MockEmbedder:
        def embed(self, texts):
            # deterministic pseudo-embeddings
            import hashlib
            res = []
            for t in texts:
                h = hashlib.sha256(t.encode("utf-8")).digest()
                res.append([b / 255.0 for b in h[:64]])
            return res

    sample_raw = [
        {
            "subject": "Alice",
            "predicate": "founded and led Acme Corp in 2019 and later sold it in 2021",
            "object": "Acme Corp",
            "time": None,
            "location": None,
            "extra": {}
        },
        {
            "subject": "Bob",
            "predicate": "joined Acme Corp in 2020",
            "object": "Acme Corp",
            "time": "2020",
            "location": None,
            "extra": {}
        }
    ]

    llm = MockLLM()
    emb = MockEmbedder()
    processed = postprocess_claims(sample_raw, llm, embedder=emb)
    print(json.dumps(processed, indent=2, ensure_ascii=False))
```

---

### Usage notes and integration checklist
- Place `prompts/claim_extraction_v2.json` in your prompts folder and wire `settings.yaml` to reference it.  
- Drop `postprocess.py` into `graphrag-extraction` and call `postprocess_claims` after raw LLM extraction. Provide real `LLMAdapter` and `EmbedderAdapter` instances.  
- Add unit tests for `normalize_predicate`, `parse_time_expression`, `needs_repair`, and `score_claim` using a small gold set.  
- Tune `min_score` and deduplication thresholds on your dataset; extend predicate normalization map and repair prompt for domain specifics.

If you want, I will next generate `grouping_config.yaml` and `grouping_demo.py` ready to paste.

# Q4:

ok

### grouping_config.yaml

```yaml
# grouping_config.yaml
# Configuration for fact grouping pipeline (GraphRAG v2)
multi_channel:
  # weights used when combining channels (subject/object/claim)
  subject_weight: 0.25
  object_weight: 0.25
  claim_weight: 0.5
  # embedding concat vs weighted-avg: "concat" or "weighted_avg"
  combine_mode: "concat"

clustering:
  # HDBSCAN parameters
  hdbscan_min_cluster_size: 6
  hdbscan_min_samples: 1
  # distance metric for HDBSCAN (euclidean recommended for concatenated vectors)
  hdbscan_metric: "euclidean"
  # fallback KMeans for splitting large clusters
  large_cluster_threshold: 200
  kmeans_split_factor: 0.5  # k = max(2, int(sqrt(n * kmeans_split_factor)))

refinement:
  # graph refinement thresholds
  graph_similarity_threshold: 0.75   # cosine threshold to create an edge
  subject_overlap_bonus: 0.15       # add to edge weight if subject identical
  object_overlap_bonus: 0.15
  time_overlap_bonus: 0.05

merge:
  # when to merge two fact groups
  summary_similarity_threshold: 0.82
  entity_overlap_threshold: 0.4

split:
  # when to split a fact group
  max_claims_per_group: 120
  internal_variance_threshold: 0.25

embedding:
  # embedding provider settings (adapter name, batch size)
  adapter: "sentence_transformers"
  batch_size: 64
  dim_claim: 384
  dim_entity: 384

runtime:
  # streaming vs batch
  mode: "batch"
  # parallel workers for embedding/clustering
  workers: 4
  random_seed: 42
```

---

### grouping_demo.py

```python
"""
grouping_demo.py
Minimal, self-contained demo of the GraphRAG v2 fact grouping pipeline.

Features:
- multi-channel claim embedding (claim sentence, subject, object)
- HDBSCAN clustering
- KMeans split for large clusters
- graph-based refinement (connected components)
- returns FactGroup objects (id, claim_ids, entity_ids placeholder)

Notes:
- Replace MockEmbedder with production EmbedderAdapter (SentenceTransformers, OpenAI, etc.)
- Requires: numpy, sklearn, hdbscan, networkx
"""

import uuid
import yaml
import numpy as np
from typing import List, Dict, Any, Tuple
from dataclasses import dataclass, field

# Optional libs: pip install hdbscan networkx scikit-learn
try:
    import hdbscan
    from sklearn.cluster import KMeans
    from sklearn.metrics.pairwise import cosine_similarity
    import networkx as nx
except Exception as e:
    raise ImportError("Install hdbscan, scikit-learn, networkx: pip install hdbscan scikit-learn networkx") from e

# ---------------- Data classes ----------------

@dataclass
class Claim:
    claim_id: str
    subject: str
    predicate: str
    object: str
    time: str = None
    location: str = None
    extra: Dict[str, Any] = field(default_factory=dict)

@dataclass
class FactGroup:
    fact_group_id: str
    claim_ids: List[str]
    entity_ids: List[str] = field(default_factory=list)
    summary_concise: str = None
    summary_detailed: str = None
    metadata: Dict[str, Any] = field(default_factory=dict)

# ---------------- Mock embedder (replace in prod) ----------------

class MockEmbedder:
    def __init__(self, dim=384):
        self.dim = dim

    def embed(self, texts: List[str]) -> List[List[float]]:
        # deterministic pseudo-embedding using hash
        import hashlib
        out = []
        for t in texts:
            h = hashlib.sha256(t.encode("utf-8")).digest()
            vec = np.array([b / 255.0 for b in h])
            if len(vec) < self.dim:
                vec = np.pad(vec, (0, self.dim - len(vec)), 'constant')
            else:
                vec = vec[:self.dim]
            out.append(vec.tolist())
        return out

# ---------------- Helpers ----------------

def claim_to_sentence(c: Claim) -> str:
    s = f"{c.subject} {c.predicate} {c.object}."
    if c.time:
        s += f" Time: {c.time}."
    if c.location:
        s += f" Location: {c.location}."
    return s

def embed_claim_multi_channel(claims: List[Claim], embedder, cfg: Dict) -> np.ndarray:
    # produce concatenated or weighted vectors per config
    claim_texts = [claim_to_sentence(c) for c in claims]
    subjects = [c.subject or "" for c in claims]
    objects = [c.object or "" for c in claims]

    v_claim = np.array(embedder.embed(claim_texts))
    v_sub = np.array(embedder.embed(subjects))
    v_obj = np.array(embedder.embed(objects))

    mode = cfg["multi_channel"].get("combine_mode", "concat")
    if mode == "concat":
        emb = np.concatenate([v_claim, v_sub, v_obj], axis=1)
    else:
        # weighted average
        sw = cfg["multi_channel"].get("subject_weight", 0.25)
        ow = cfg["multi_channel"].get("object_weight", 0.25)
        cw = cfg["multi_channel"].get("claim_weight", 0.5)
        emb = v_claim * cw + v_sub * sw + v_obj * ow
    return emb

# ---------------- Clustering ----------------

def cluster_claims_hdbscan(emb: np.ndarray, cfg: Dict) -> List[int]:
    min_cluster_size = cfg["clustering"].get("hdbscan_min_cluster_size", 6)
    min_samples = cfg["clustering"].get("hdbscan_min_samples", 1)
    metric = cfg["clustering"].get("hdbscan_metric", "euclidean")
    clusterer = hdbscan.HDBSCAN(min_cluster_size=min_cluster_size, min_samples=min_samples, metric=metric)
    labels = clusterer.fit_predict(emb)
    return labels.tolist()

def split_large_clusters(labels: List[int], emb: np.ndarray, cfg: Dict) -> List[int]:
    labels = labels.copy()
    threshold = cfg["clustering"].get("large_cluster_threshold", 200)
    n = len(labels)
    unique = set(labels)
    for lab in unique:
        if lab == -1:
            continue
        idxs = [i for i, l in enumerate(labels) if l == lab]
        if len(idxs) > threshold:
            k = max(2, int(np.sqrt(len(idxs) * cfg["clustering"].get("kmeans_split_factor", 0.5))))
            sub_emb = emb[idxs]
            km = KMeans(n_clusters=k, random_state=cfg.get("runtime", {}).get("random_seed", 42)).fit(sub_emb)
            for i_local, global_idx in enumerate(idxs):
                labels[global_idx] = f"{lab}_split_{km.labels_[i_local]}"
    return labels

# ---------------- Graph refinement ----------------

def refine_with_graph(claims: List[Claim], emb: np.ndarray, labels: List[int], cfg: Dict) -> List[FactGroup]:
    # Build initial groups from labels (ignore -1 as noise)
    groups = {}
    for idx, lab in enumerate(labels):
        if lab == -1:
            continue
        groups.setdefault(str(lab), []).append(idx)

    # Build similarity graph among claims
    sim = cosine_similarity(emb)
    G = nx.Graph()
    n = len(claims)
    for i in range(n):
        G.add_node(i)
    thresh = cfg["refinement"].get("graph_similarity_threshold", 0.75)
    subj_bonus = cfg["refinement"].get("subject_overlap_bonus", 0.15)
    obj_bonus = cfg["refinement"].get("object_overlap_bonus", 0.15)
    time_bonus = cfg["refinement"].get("time_overlap_bonus", 0.05)

    for i in range(n):
        for j in range(i + 1, n):
            weight = float(sim[i, j])
            if claims[i].subject and claims[j].subject and claims[i].subject == claims[j].subject:
                weight += subj_bonus
            if claims[i].object and claims[j].object and claims[i].object == claims[j].object:
                weight += obj_bonus
            if claims[i].time and claims[j].time and claims[i].time == claims[j].time:
                weight += time_bonus
            if weight >= thresh:
                G.add_edge(i, j, weight=weight)

    # For each initial label, take connected components within that label's nodes
    fact_groups = []
    for lab, idxs in groups.items():
        subG = G.subgraph(idxs)
        comps = list(nx.connected_components(subG))
        for comp in comps:
            claim_ids = [claims[i].claim_id for i in comp]
            fg = FactGroup(fact_group_id=str(uuid.uuid4()), claim_ids=claim_ids)
            fact_groups.append(fg)

    # Noise handling: optionally create small groups for -1 or attach to nearest group
    noise_idxs = [i for i, l in enumerate(labels) if l == -1]
    for ni in noise_idxs:
        # attach to nearest group by max similarity
        best_fg = None
        best_score = 0.0
        for fg in fact_groups:
            # compute average sim to group
            idxs = [next(i for i,c in enumerate(claims) if c.claim_id==cid) for cid in fg.claim_ids]
            avg = float(np.mean([sim[ni, j] for j in idxs]))
            if avg > best_score:
                best_score = avg
                best_fg = fg
        if best_fg and best_score >= cfg["refinement"].get("graph_similarity_threshold", 0.75):
            best_fg.claim_ids.append(claims[ni].claim_id)
        else:
            # create new small group
            fact_groups.append(FactGroup(fact_group_id=str(uuid.uuid4()), claim_ids=[claims[ni].claim_id]))

    return fact_groups

# ---------------- High-level pipeline ----------------

def build_fact_groups(claims: List[Claim], embedder, cfg: Dict) -> List[FactGroup]:
    if not claims:
        return []
    emb = embed_claim_multi_channel(claims, embedder, cfg)
    labels = cluster_claims_hdbscan(emb, cfg)
    labels = split_large_clusters(labels, emb, cfg)
    fgs = refine_with_graph(claims, emb, labels, cfg)
    # optional merge/split heuristics can be applied here (not implemented in demo)
    return fgs

# ---------------- Demo main ----------------

def demo():
    # load config
    cfg = yaml.safe_load(open("grouping_config.yaml", "r", encoding="utf-8"))
    # sample claims
    claims = [
        Claim(claim_id="c1", subject="NVIDIA", predicate="released", object="Blackwell architecture", time="2024"),
        Claim(claim_id="c2", subject="NVIDIA", predicate="announced", object="Blackwell GPUs", time="2024"),
        Claim(claim_id="c3", subject="Alice", predicate="founded", object="Acme Corp", time="2019"),
        Claim(claim_id="c4", subject="Bob", predicate="joined", object="Acme Corp", time="2020"),
        Claim(claim_id="c5", subject="Acme Corp", predicate="was acquired by", object="MegaCorp", time="2021"),
    ]
    embedder = MockEmbedder(dim=128)  # replace with real embedder
    fgs = build_fact_groups(claims, embedder, cfg)
    print("Fact groups:")
    for fg in fgs:
        print(f"- FG {fg.fact_group_id}: claims={fg.claim_ids}")

if __name__ == "__main__":
    demo()
```

---

### 使用說明與整合要點（簡短）
- 把 `grouping_config.yaml` 放到 `config/`，把 `grouping_demo.py` 放到 `packages/graphrag-grouping/`。  
- 在生產環境把 `MockEmbedder` 換成 `EmbedderAdapter`（SentenceTransformersAdapter 或 OpenAI embedder）。  
- 調整 `combine_mode` 與權重以配合實際 embedding 維度（若使用 concat，index/metric 需支援更高維度）。  
- 在整合時，將 `build_fact_groups` 的輸出 `FactGroup` 與 storage/provenance 層連接，並在 summarizer 中使用 `claim_ids` 生成 concise/detailed summaries。  

如果你要我，我可以接著產出：
- `grouping_demo.py` 的 production 版本（替換 embedder、加入 merge/split heuristics、unit tests），或  
- 一個 `autotune.py` 範例，用於自動調整 HDBSCAN / merge thresholds 並輸出 tuning report。