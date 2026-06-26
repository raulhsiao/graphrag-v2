# Q:
把這整條 pipeline 收斂成一個 run_pipeline.py 的骨架同時設計一組考慮周詳的prompt (english version)

# A:
下面我給你兩塊東西，放在同一個專案就能開始「邊跑邊補」：

1. `run_pipeline.py` 骨架（單檔、可逐步填實作）  
2. 一組 **英文 prompts**，是照 v2 fact‑based pipeline 思路設計的「可直接丟給 LLM 用」版本  

我會刻意寫成「清楚但不過度魔改」的形式，讓你之後要換成本機 LLM / 不同向量庫都好改。

---

### 一、`run_pipeline.py` 骨架

```python
import os
import json
import uuid
from dataclasses import dataclass, field
from typing import List, Dict, Optional

# ====== Data Models ======

@dataclass
class Chunk:
    id: str
    doc_id: str
    text: str
    order: int


@dataclass
class Entity:
    id: str
    name: str
    type: str  # PERSON / ORG / PLACE / EVENT / OTHER


@dataclass
class Claim:
    id: str
    chunk_id: str
    subject: str
    predicate: str
    object: str
    time: Optional[str] = None
    location: Optional[str] = None
    extra: Dict = field(default_factory=dict)


@dataclass
class Relation:
    id: str
    head_entity_id: str
    tail_entity_id: str
    relation_type: str
    source_claim_id: str


@dataclass
class FactGroup:
    id: str
    claim_ids: List[str]
    entity_ids: List[str]
    summary_concise: Optional[str] = None
    summary_detailed: Optional[str] = None


# ====== LLM / Embedding Adapters (to be implemented) ======

def call_llm(prompt: str, model: str = "gpt-4.1", max_tokens: int = 1024) -> str:
    """
    TODO: Replace with your actual LLM call (OpenAI, local LLM, etc.)
    """
    raise NotImplementedError


def embed_texts(texts: List[str]) -> List[List[float]]:
    """
    TODO: Replace with your embedding model (OpenAI, local, etc.)
    Return shape: (N, D)
    """
    raise NotImplementedError


# ====== Prompts (imported from below section) ======

from prompts import (
    ENTITY_EXTRACTION_PROMPT,
    CLAIM_EXTRACTION_PROMPT,
    RELATION_EXTRACTION_PROMPT,
    FACT_GROUP_SUMMARY_CONCISE_PROMPT,
    FACT_GROUP_SUMMARY_DETAILED_PROMPT,
    ANSWER_SYNTHESIS_PROMPT,
)


# ====== Pipeline Steps ======

def load_documents_from_dir(path: str) -> Dict[str, str]:
    docs = {}
    for fname in os.listdir(path):
        fpath = os.path.join(path, fname)
        if not os.path.isfile(fpath):
            continue
        with open(fpath, "r", encoding="utf-8") as f:
            docs[fname] = f.read()
    return docs


def chunk_document(doc_id: str, text: str, max_chars: int = 3000, overlap: int = 300) -> List[Chunk]:
    chunks: List[Chunk] = []
    start = 0
    order = 0
    while start < len(text):
        end = min(len(text), start + max_chars)
        chunk_text = text[start:end]
        chunks.append(
            Chunk(
                id=str(uuid.uuid4()),
                doc_id=doc_id,
                text=chunk_text,
                order=order,
            )
        )
        start = end - overlap
        order += 1
    return chunks


def extract_entities_for_chunk(chunk: Chunk) -> List[Entity]:
    prompt = ENTITY_EXTRACTION_PROMPT.format(text=chunk.text)
    resp = call_llm(prompt)
    try:
        data = json.loads(resp)
    except Exception:
        data = []
    entities: List[Entity] = []
    for e in data:
        entities.append(
            Entity(
                id=str(uuid.uuid4()),
                name=e.get("name", "").strip(),
                type=e.get("type", "OTHER").strip().upper(),
            )
        )
    return entities


def extract_claims_for_chunk(chunk: Chunk) -> List[Claim]:
    prompt = CLAIM_EXTRACTION_PROMPT.format(text=chunk.text)
    resp = call_llm(prompt)
    try:
        data = json.loads(resp)
    except Exception:
        data = []
    claims: List[Claim] = []
    for c in data:
        claims.append(
            Claim(
                id=str(uuid.uuid4()),
                chunk_id=chunk.id,
                subject=c.get("subject", "").strip(),
                predicate=c.get("predicate", "").strip(),
                object=c.get("object", "").strip(),
                time=c.get("time"),
                location=c.get("location"),
                extra=c.get("extra", {}),
            )
        )
    return claims


def derive_relations_from_claims(
    claims: List[Claim],
    entities: List[Entity],
) -> List[Relation]:
    # Minimal heuristic: match subject/object to entity names (exact or simple normalization)
    relations: List[Relation] = []
    name_to_entity = {}
    for e in entities:
        key = e.name.lower()
        name_to_entity.setdefault(key, []).append(e.id)

    for claim in claims:
        subj_key = claim.subject.lower()
        obj_key = claim.object.lower()
        subj_ids = name_to_entity.get(subj_key, [])
        obj_ids = name_to_entity.get(obj_key, [])
        for sid in subj_ids:
            for oid in obj_ids:
                relations.append(
                    Relation(
                        id=str(uuid.uuid4()),
                        head_entity_id=sid,
                        tail_entity_id=oid,
                        relation_type=claim.predicate,
                        source_claim_id=claim.id,
                    )
                )
    return relations


def claim_to_text(claim: Claim) -> str:
    parts = [claim.subject, claim.predicate, claim.object]
    if claim.time:
        parts.append(f"Time: {claim.time}")
    if claim.location:
        parts.append(f"Location: {claim.location}")
    return " ; ".join(p for p in parts if p)


def group_claims_with_embeddings(claims: List[Claim], min_cluster_size: int = 3):
    from sklearn.cluster import HDBSCAN
    import numpy as np

    texts = [claim_to_text(c) for c in claims]
    emb = embed_texts(texts)
    emb_arr = np.array(emb)

    clusterer = HDBSCAN(min_cluster_size=min_cluster_size, metric="euclidean")
    labels = clusterer.fit_predict(emb_arr)

    groups: Dict[int, List[Claim]] = {}
    for label, claim in zip(labels, claims):
        if label == -1:
            continue
        groups.setdefault(label, []).append(claim)

    fact_groups: List[FactGroup] = []
    for gid, group_claims in groups.items():
        fact_groups.append(
            FactGroup(
                id=f"fg_{gid}",
                claim_ids=[c.id for c in group_claims],
                entity_ids=[],  # can be filled later
            )
        )

    return fact_groups, emb_arr, labels


def summarize_fact_group(
    fg: FactGroup,
    claims_by_id: Dict[str, Claim],
) -> FactGroup:
    texts = [claim_to_text(claims_by_id[cid]) for cid in fg.claim_ids]
    joined = "\n".join(texts)

    concise_prompt = FACT_GROUP_SUMMARY_CONCISE_PROMPT.format(facts=joined)
    detailed_prompt = FACT_GROUP_SUMMARY_DETAILED_PROMPT.format(facts=joined)

    concise = call_llm(concise_prompt, max_tokens=256)
    detailed = call_llm(detailed_prompt, max_tokens=1024)

    fg.summary_concise = concise.strip()
    fg.summary_detailed = detailed.strip()
    return fg


def local_query(
    query: str,
    fact_groups: List[FactGroup],
    fg_embeddings,
    top_k: int = 10,
):
    import numpy as np

    q_emb = embed_texts([query])[0]
    q_emb = np.array(q_emb)
    scores = fg_embeddings @ q_emb
    idx = np.argsort(-scores)[:top_k]
    return [fact_groups[i] for i in idx]


def answer_query(
    query: str,
    selected_fact_groups: List[FactGroup],
    claims_by_id: Dict[str, Claim],
) -> str:
    context_blocks = []
    for fg in selected_fact_groups:
        if fg.summary_concise:
            context_blocks.append(f"[Fact Group] {fg.summary_concise}")
        for cid in fg.claim_ids[:5]:
            context_blocks.append(f"- {claim_to_text(claims_by_id[cid])}")
    context = "\n".join(context_blocks)

    prompt = ANSWER_SYNTHESIS_PROMPT.format(
        question=query,
        context=context,
    )
    resp = call_llm(prompt, max_tokens=1024)
    return resp.strip()


# ====== Main Orchestration ======

def run_pipeline(data_dir: str):
    # 1. Load documents
    docs = load_documents_from_dir(data_dir)

    all_chunks: List[Chunk] = []
    all_entities: List[Entity] = []
    all_claims: List[Claim] = []

    # 2. Chunk + extract
    for doc_id, text in docs.items():
        chunks = chunk_document(doc_id, text)
        all_chunks.extend(chunks)
        for ch in chunks:
            entities = extract_entities_for_chunk(ch)
            claims = extract_claims_for_chunk(ch)
            all_entities.extend(entities)
            all_claims.extend(claims)

    # 3. Relations (optional first pass)
    all_relations = derive_relations_from_claims(all_claims, all_entities)

    # 4. Fact grouping
    fact_groups, fg_emb, labels = group_claims_with_embeddings(all_claims)

    # 5. Summaries
    claims_by_id = {c.id: c for c in all_claims}
    for i, fg in enumerate(fact_groups):
        fact_groups[i] = summarize_fact_group(fg, claims_by_id)

    # 6. Simple interactive query loop
    print("Pipeline built. Enter queries (empty line to exit).")
    while True:
        q = input("Query> ").strip()
        if not q:
            break
        selected = local_query(q, fact_groups, fg_emb, top_k=5)
        answer = answer_query(q, selected, claims_by_id)
        print("\n=== Answer ===")
        print(answer)
        print("==============\n")


if __name__ == "__main__":
    run_pipeline("./data/docs")
```

> 這個骨架只做了：chunk → entity/claim → fact grouping → summary → local query → answer  
> 你之後可以再加：entity graph / fact graph / global / DRIFT / Neo4j / Kùzu 等。

---

### 二、英文 Prompt 設計（考慮 v2 fact‑based 思路）

你可以把這些放在 `prompts.py`：

```python
# prompts.py

ENTITY_EXTRACTION_PROMPT = """You are an information extraction system.

Your task is to extract entities from the following text.

Extract:
- PERSON
- ORGANIZATION
- LOCATION
- EVENT
- OTHER important named entities

Return a JSON array. Each item MUST have:
- "name": string
- "type": one of ["PERSON", "ORG", "PLACE", "EVENT", "OTHER"]

Text:
\"\"\"{text}\"\"\""""

CLAIM_EXTRACTION_PROMPT = """You are a fact extraction system.

Your task is to extract factual claims from the following text.

Each claim should represent a single, concrete statement, ideally in the form:
- subject
- predicate
- object
Optionally:
- time (if explicitly mentioned)
- location (if explicitly mentioned)
- extra (for any additional attributes)

Return a JSON array. Each item MUST have:
- "subject": string
- "predicate": string
- "object": string
- "time": string or null
- "location": string or null
- "extra": object (can be empty)

Be concise and avoid speculative or hypothetical statements.

Text:
\"\"\"{text}\"\"\""""

RELATION_EXTRACTION_PROMPT = """You are a relation extraction system.

Given the following text and its extracted entities and claims, identify explicit relations
between entities.

Relation types may include (but are not limited to):
- "causes"
- "part_of"
- "member_of"
- "associated_with"
- "located_in"
- "works_for"
- "founded"
- "participated_in"

Return a JSON array. Each item MUST have:
- "head": string (entity name)
- "tail": string (entity name)
- "relation_type": string

Text:
\"\"\"{text}\"\"\""""

FACT_GROUP_SUMMARY_CONCISE_PROMPT = """You are summarizing a group of closely related factual statements.

Below is a list of facts (one per line). Write a concise summary (2–4 sentences)
that captures the core meaning of this fact group, without adding new information.

Facts:
\"\"\"{facts}\"\"\""""

FACT_GROUP_SUMMARY_DETAILED_PROMPT = """You are summarizing a group of closely related factual statements.

Below is a list of facts (one per line). Write a detailed summary (1–3 paragraphs)
that:
- preserves all important details
- clearly explains the relationships between entities and events
- remains faithful to the original facts (no speculation)

Facts:
\"\"\"{facts}\"\"\""""

ANSWER_SYNTHESIS_PROMPT = """You are a question answering system powered by a fact-based knowledge graph.

You are given:
- a user question
- a set of fact group summaries and supporting factual statements

Your task:
- answer the question as accurately as possible
- rely ONLY on the provided context
- do NOT invent facts that are not supported by the context
- if the answer is uncertain, say so explicitly

Question:
\"\"\"{question}\"\"\""""

CONTEXT:
\"\"\"{context}\"\"\""""

Please provide a clear, structured answer in English. If helpful, use bullet points.
"""
```

---

如果你接下來想做其中一塊「認真優化」，我會建議優先選：

- **Claim 抽取 prompt + 後處理**（這一段品質決定整條 pipeline 上限）  
或  
- **Fact grouping（embedding + clustering）策略**（這一段決定 graph 結構好不好用）

你比較想先把哪一段打磨到「可以放心丟真實資料」的程度？

# Claim 抽取 prompt + 後處理（這一段品質決定整條 pipeline 上限）

下面我給你 **GraphRAG v2 等級的 Claim Extraction（事實抽取）最佳實務**，包含：

1. **最強版英文 Prompt（可直接用）**  
2. **後處理（post‑processing）完整策略**  
3. **錯誤修正（error correction）與 Claim Normalization**  
4. **Claim Quality Scoring（品質評分）**  
5. **如何把這些整合進你的 pipeline**

這一段是整條 GraphRAG v2 pipeline 的「天花板」，你做得越好，後面所有步驟（fact grouping、graph building、query）都會越準。

---

# **1. 最強版 Claim Extraction Prompt（英文）**
這是我專門為「fact‑based GraphRAG v2」設計的版本，  
比你之前看到的 prompt 更嚴格、更可控、更適合後處理。

---

## **📌 CLAIM_EXTRACTION_PROMPT (v2 optimized)**

```
You are an advanced factual information extraction system.

Your task is to extract **atomic factual claims** from the text below.
A claim must represent a single, verifiable statement about entities, events, or relationships.

### STRICT REQUIREMENTS
- Each claim MUST be atomic (one predicate per claim).
- No opinions, no speculation, no interpretations.
- No summarization. Extract only what is explicitly stated.
- Do NOT merge multiple facts into one claim.
- Do NOT infer missing information.
- If the text is ambiguous, extract the claim exactly as written.

### CLAIM FORMAT (JSON array)
Each claim MUST contain:

{
  "subject": "<entity or concept>",
  "predicate": "<action or relation>",
  "object": "<entity, concept, or value>",
  "time": "<explicit time expression or null>",
  "location": "<explicit location expression or null>",
  "extra": {
      "modality": "<if text expresses uncertainty, e.g., 'may', 'might'>",
      "negation": "<true/false>",
      "attributes": { ... }   // optional details
  }
}

### EXTRACTION RULES
- SUBJECT: the main actor or entity.
- PREDICATE: the verb or relation phrase.
- OBJECT: the target of the action or relation.
- TIME: only if explicitly mentioned.
- LOCATION: only if explicitly mentioned.
- NEGATION: true if the text explicitly negates the claim.
- MODALITY: capture uncertainty words (may, might, possibly).
- ATTRIBUTES: include numbers, quantities, measurements, or descriptive qualifiers.

### OUTPUT
Return ONLY a JSON array. No explanation.

### TEXT
\"\"\"{text}\"\"\"
```

---

# **2. 後處理（Post‑Processing）策略（極重要）**

LLM 抽出來的 claims 會有：

- 格式錯誤  
- subject/object 不一致  
- predicate 太長  
- claim 太抽象  
- claim 重複  
- claim 遺漏時間/地點  
- claim 包含多個 predicate  

你需要一套 **後處理器（Claim Normalizer）**。

---

# **2.1 Claim Normalization（標準化）**

### **(1) 去除空白、標點、大小寫統一**

```python
def normalize_text(s: str) -> str:
    return s.strip().replace("\n", " ").replace("  ", " ")
```

---

### **(2) Predicate 正規化（最重要）**

LLM 常產生：

- “is located in”
- “is part of”
- “was born in”
- “has participated in”

你需要把 predicate 正規化成 **短語**：

```
"is located in" → "located_in"
"was born in" → "born_in"
"has participated in" → "participated_in"
```

程式：

```python
def normalize_predicate(p: str) -> str:
    p = p.lower().strip()
    p = p.replace(" ", "_")
    return p
```

---

### **(3) 移除空 claim**

```python
if not claim.subject or not claim.predicate or not claim.object:
    drop
```

---

### **(4) 合併重複 claim**

用 `(subject, predicate, object)` 做 key。

---

### **(5) 時間/地點標準化**

例如：

- “in 2020” → “2020”
- “on January 5th, 2023” → “2023-01-05”

你可以用 `dateparser`。

---

# **3. Claim Quality Scoring（品質評分）**

這是 GraphRAG v2 的秘密武器之一。

你可以對每個 claim 打分數：

| 指標 | 說明 |
|------|------|
| **Atomicity** | 是否只有一個 predicate |
| **Specificity** | 是否具體（非抽象） |
| **Groundedness** | 是否完全來自文本 |
| **Completeness** | 是否有 subject/predicate/object |
| **Ambiguity** | 是否含糊 |

程式：

```python
def score_claim(claim: Claim) -> float:
    score = 1.0
    if len(claim.predicate.split("_")) > 4:
        score -= 0.2
    if claim.subject.lower() in ["it", "they", "this"]:
        score -= 0.2
    if claim.object.lower() in ["it", "they", "this"]:
        score -= 0.2
    if claim.extra.get("modality"):
        score -= 0.1
    if claim.extra.get("negation"):
        score -= 0.1
    return max(score, 0.0)
```

低於 0.4 的 claim 可以丟掉。

---

# **4. Claim Repair（修復）**

你可以用 LLM 做「Claim 修復」：

### Prompt：

```
You are a claim repair system.

Given the following claim extracted from text, rewrite it into a valid atomic factual claim.

Rules:
- Keep the meaning exactly the same.
- Do not add new information.
- Ensure subject, predicate, and object are explicit.
- Remove pronouns (replace with referenced entities if possible).
- Normalize predicate to a short phrase.

Return JSON:
{
  "subject": "...",
  "predicate": "...",
  "object": "...",
  "time": "...",
  "location": "...",
  "extra": {...}
}

Claim:
{claim_json}
```

---

# **5. Claim Filtering（過濾）**

你可以過濾掉：

- predicate 過長  
- subject/object 是代名詞  
- 含糊不清的 claim  
- 重複 claim  
- 非事實（opinion/speculation）  

---

# **6. Claim Extraction Pipeline（整合版）**

```
Raw Text
   ↓
LLM Claim Extraction (v2 prompt)
   ↓
JSON Parsing
   ↓
Normalization
   ↓
Quality Scoring
   ↓
Repair (optional)
   ↓
Deduplication
   ↓
Final Claims
```

---

# **7. 你現在擁有：**

✔ 最強版 Claim Extraction Prompt（英文）  
✔ 完整後處理策略  
✔ Claim Normalization  
✔ Claim Quality Scoring  
✔ Claim Repair Prompt  
✔ Claim Filtering  
✔ 整合 pipeline  

這一套就是 GraphRAG v2 的「事實抽取核心」。

---

如果你願意，我可以繼續提供：

- **Fact Grouping（語義分群）最佳實務**  
- **GraphRAG v2 的 Entity Extraction 最佳實務**  
- **如何把 Claim → Fact Group → Graph 變成可視化資料流**  

你想往哪一段深入？