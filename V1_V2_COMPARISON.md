# GraphRAG v1（community-leading）vs v2（fact-leading）比較報告

> 本報告比較 GraphRAG v1（`indexing_method: standard`）與本分支實作的
> v2 fact-leading（`indexing_method: fact-leading`）。
> 兩者共用同一套程式碼基礎，僅以 settings.yaml 切換。

---

## 1. 一句話總結

| 版本 | 核心思想 |
|---|---|
| **v1 community-leading** | 先建**實體圖**（entity graph），再對「圖的結構」分群 → 社群（community）是知識單位 |
| **v2 fact-leading** | 先抽**事實**（facts / claims），再對「事實本身」分群 → 事實群組（fact group）是知識單位 |

---

## 2. 設計哲學比較

| 項目 | v1 community-leading | v2 fact-leading |
|---|---|---|
| 最小知識單位 | chunk（text unit）＋ entity | **fact（原子事實 claim）** |
| 中階組織單位 | community（圖論社群） | **fact group（事實群組）** |
| 分群的對象 | 實體關係圖（誰連誰） | 事實圖（哪些事實講同一件事） |
| 分群依據 | 圖的**拓撲結構**（關係密度） | 事實間的**內容關聯**（共享實體、可選語意相似度） |
| 報告描述什麼 | 「這群實體之間的關係網絡」 | 「這組事實共同構成的敘事」 |
| 溯源（provenance） | community → entity → text_unit | **fact group → claim → text_unit**（每句事實可回溯原文） |

---

## 3. Pipeline 流程對照

```
v1 standard                          v2 fact-leading
─────────────────────────            ─────────────────────────
load_input_documents                 load_input_documents
create_base_text_units               create_base_text_units      ← chunk（兩者相同）
create_final_documents               create_final_documents
extract_graph                        extract_graph               ← 實體/關係抽取（兩者相同）
finalize_graph                       finalize_graph
extract_covariates (選配)            extract_covariates (強制)   ← 事實抽取＝v2 核心
create_communities        ◄──差異──► create_fact_groups          ← 分群方式不同
create_final_text_units              create_final_text_units
create_community_reports  ◄──差異──► create_fact_group_reports   ← 報告內容不同
generate_text_embeddings             generate_text_embeddings    ← lancedb（兩者相同）
```

**只有兩個階段被替換**，其餘完全共用 — 這是「在原架構上附加」的設計原則。

---

## 4. 功能／特性逐項比較

| # | 比較項目 | v1 community-leading | v2 fact-leading |
|---|---|---|---|
| 1 | **community / fact group 怎麼產生** | 對「實體關係圖」跑 hierarchical Leiden：節點=實體、邊=LLM 抽出的關係 | 對「事實圖」跑 hierarchical Leiden：節點=fact、邊=**共享實體鏈**（同主詞/受詞的事實相連）＋可選 embedding 相似邊 |
| 2 | **是否需要 chunk** | 需要。chunk 是抽取單位與檢索證據 | **仍然需要**。fact 從 chunk 抽出並記錄 `text_unit_id`；查詢引用時回溯到 chunk 原文 |
| 3 | **是否需要實體抽取** | 需要（圖的節點） | 需要（用來連接事實、解析 entity_ids、供 local search 使用） |
| 4 | **claims 抽取** | 選配（`extract_claims.enabled: false` 預設） | **強制開啟**（fact 是主體，設定會自動改為 true） |
| 5 | **分群上限** | `cluster_graph.max_cluster_size: 10`（每群最多實體數） | `fact_grouping.max_group_size: 10`（每群最多事實數） |
| 6 | **孤立節點處理** | `use_lcc: true` 預設：只保留最大連通分量，圖外實體不成社群 | `use_lcc: false` 預設＋**singleton 保證**：每個 fact 一定屬於某個 level-0 群組 |
| 7 | **階層結構** | 有（Leiden levels，parent / children） | 有（同一機制，直接沿用） |
| 8 | **報告 prompt 的輸入** | 實體表＋關係表＋（選配）claims 表 | **facts CSV**（id, subject, type, description, source）|
| 9 | **報告引用格式** | `[Data: Reports/Entities/Relationships (ids)]` | `[Data: Facts (ids)]` |
| 10 | **輸出 parquet** | documents, text_units, entities, relationships, (covariates), communities, community_reports | 同左（covariates 必有）＋**新增 `fact_groups.parquet`**（每群的 claim_ids 溯源） |
| 11 | **向量庫（lancedb）** | entity_description / community_full_content / text_unit_text | **完全相同**（community_full_content 內容變為 fact group 報告） |
| 12 | **查詢模式** | local / global / drift / basic | **四種全部可用、程式不需修改**（吃相同的表） |
| 13 | **增量更新（update）** | 支援（standard-update / fast-update） | 尚未支援（會明確報錯） |
| 14 | **Prompt auto-tuning** | 支援（extract_graph 等） | fact prompts 可手動替換檔案；auto-tuning 尚未涵蓋 fact prompt |
| 15 | **LLM 成本** | 抽取：實體/關係＋社群報告 | **較高**：多一道 claims 抽取（每 chunk 一次以上，含 gleanings） |
| 16 | **確定性** | Leiden 固定 seed，可重現 | 同左（共用 seed 設定），分群本身**不用 LLM**，完全可重現 |
| 17 | **適用場景** | 主題總覽、關係網絡分析、「這個資料集在講什麼」 | 敘事/事件追蹤、需要逐句證據回溯、合規/法務/調查型問答 |

---

## 5. 設定切換方式（settings.yaml 一鍵切換）

```yaml
# v1 community-leading（預設）
indexing_method: standard      # 或 fast（NLP 快速版）

# v2 fact-leading
indexing_method: fact-leading

fact_grouping:
  max_group_size: 10           # 每個事實群組的最大事實數（Leiden 上限）
  use_text_similarity: false   # true 時額外用 embedding 相似度連接事實
  similarity_threshold: 0.75   # 相似邊的 cosine 門檻
  # use_lcc: false             # 事實圖較零散，預設不限制連通分量
  # seed: 3735928559           # 分群固定種子，確保可重現

fact_group_reports:
  completion_model_id: default_completion_model
  prompt: "prompts/fact_group_report.txt"
  max_length: 2000             # 報告輸出上限（tokens）
  max_input_length: 8000       # 報告輸入 facts 上限（tokens）
```

CLI 不帶 `--method` 時以 settings.yaml 為準；帶了則覆寫。

---

## 6. Fact 抽取的實作說明（本分支的做法）

### 6.1 抽取方式：與 v1 抽實體/關係「同一個模式」

v1 對每個 chunk 用 `prompts/extract_graph.txt` 抽實體與關係；
v2 的 fact 抽取採用**完全相同的模式**——對每個 chunk（text unit）用
`prompts/extract_claims.txt` 讓 LLM 抽出原子事實：

```
chunk (text unit)
   │  LLM + prompts/extract_claims.txt
   ▼
(SUBJECT<|>OBJECT<|>CLAIM_TYPE<|>STATUS<|>START_DATE<|>END_DATE<|>DESCRIPTION<|>SOURCE_TEXT)
   │  解析 + 正規化
   ▼
covariates.parquet   ← 每列一個 fact，含 text_unit_id 溯源
```

每個 fact 包含：**主詞實體、受詞實體、事實類型、狀態
（TRUE/FALSE/SUSPECTED）、起迄時間（ISO-8601）、描述、原文引句**。
抽取器支援 **gleanings 多輪補抽**（`max_gleanings`）：第一輪抽完後追問
LLM「還有沒有漏掉的事實」，降低遺漏率。

這一步**直接重用 v1 既有的 claim extractor**（`extract_covariates`
workflow），不另造輪子——v1 已具備高品質的事實抽取器，只是預設關閉且
抽出的 claims 僅作 local search 的輔助資料；v2 把它升級為 pipeline 的
**主角**：後續的分群、報告、全域摘要全部以這些 facts 為基礎。

### 6.2 抽取之後：分群不需要 LLM

facts 進入 `create_fact_groups` 後為**純演算法**（零 LLM 成本、可重現）：

1. 共享實體連邊：主詞或受詞相同的 facts 以線性鏈相連（邊數 O(n)）
2. （選配）語意連邊：fact 描述做 embedding，cosine ≥ 門檻者相連
3. 事實圖跑 hierarchical Leiden → 階層式 fact groups
4. 未入圖的 facts 補成 singleton 群組（**保證 100% 覆蓋**）

之後 `create_fact_group_reports` 才再次使用 LLM，為每個群組寫報告。

### 6.3 需要哪些 prompt 檔案？

| Prompt 檔案 | 來源 | 在 v2 中的角色 | 必要性 |
|---|---|---|---|
| `prompts/extract_claims.txt` | v1 既有 | **fact 抽取核心**：從 chunk 抽出原子事實 | **必要** |
| `prompts/extract_graph.txt` | v1 既有 | 實體/關係抽取：實體用於連接事實與查詢 context | **必要** |
| `prompts/summarize_descriptions.txt` | v1 既有 | 合併同一實體多處描述 | 必要 |
| `prompts/fact_group_report.txt` | **v2 新增** | fact group 報告：輸入 facts CSV，輸出 title/summary/findings/rating JSON，引用 `[Data: Facts (ids)]` | **必要** |
| `prompts/community_report_graph.txt` | v1 既有 | 只有 community-leading 用 | v2 不用 |
| 查詢類 prompts（local/global/drift/basic） | v1 既有 | 兩種模式共用 | 必要 |

以上檔案 `graphrag init` 會全部自動產生；fact grouping 這一步**本身不需
要任何 prompt**（純演算法）。

### 6.4 調整 fact 抽取品質的旋鈕

```yaml
extract_claims:
  enabled: true                       # fact-leading 下自動強制
  completion_model_id: default_completion_model
  prompt: "prompts/extract_claims.txt" # 可替換成領域化版本
  description: "Any claims or facts that could be relevant to information discovery."
  max_gleanings: 1                    # 補抽輪數，調高可減少遺漏（成本增加）
```

`description` 是最重要的旋鈕：它告訴 LLM「什麼算是值得抽的事實」，
換成領域描述（例如「合規風險與裁罰事件」）即可聚焦抽取方向。

---

## 7. 驗證方式

| 工具 | 涵蓋 | 執行 |
|---|---|---|
| pytest verb 測試 | fact 分群 schema/覆蓋率/階層、報告 schema | `pytest tests/verbs -k fact` |
| 端到端工具 | 設定切換、pipeline 選擇、完整建索引、parquet/lancedb、四種查詢、v1 回歸（16 項檢查，全 mock 免 API key） | `uv run python scripts/test_fact_leading.py` |
