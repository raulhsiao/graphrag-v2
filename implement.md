# 任務完成
已在 feature/fact-leading 分支提交 commit d1277c7f（21 個檔案、+1,702 行），最終驗證全數通過：pytest 320/320、端到端測試工具 16/16。

## 交付內容總覽
1. v1 功能勘查與彙整 — 完整清單寫在 FACT_LEADING.md：settings.yaml 設定介面、chunking、LLM/NLP 雙路圖譜抽取、claims 抽取、Leiden 社群分群、社群報告、三種 embedding（lancedb）、四種查詢模式、prompt auto-tuning、增量更新、快取等。

2. settings.yaml 切換介面（依 doc/ 的 fact-leading 設計）

- indexing_method: fact-leading   # community-leading 用 standard / fast
- fact_grouping:
  max_group_size: 10
  use_text_similarity: false    # 可選：以 embedding 相似度連接 facts
  similarity_threshold: 0.75
- fact_group_reports:
  prompt: "prompts/fact_group_report.txt"
- fact-leading 會自動強制開啟 extract_claims；CLI --method 未指定時以 settings.yaml 為準。

3. fact-leading pipeline（最大限度沿用原架構）

- create_fact_groups：以 facts（claims）為節點建事實圖（共享實體連邊 + 可選語意相似邊），重用既有 hierarchical Leiden 產生階層式 fact groups；每個 fact 保證歸屬 level-0 群組
- create_fact_group_reports：以每組 facts 為 context 產生報告（引用格式 [Data: Facts (ids)]）
- 輸出保持原 parquet schema（communities / community_reports）+ 新增 fact_groups.parquet 溯源表；lancedb 三個 collection 不變，因此 local/global/drift/basic 查詢無需修改即可運作
4. 測試工具

- tests/verbs/test_create_fact_groups.py、test_create_fact_group_reports.py（pytest，用官方 fixtures + mock LLM）
- scripts/test_fact_leading.py：獨立端到端工具（全 mock、免 API key），一次驗證設定切換、pipeline 選擇、完整建索引、parquet/lancedb 輸出不變量、四種查詢模式、community-leading 回歸。執行方式：uv run python scripts/test_fact_leading.py
- 一個附帶改進：為了讓查詢路徑可用 mock 測試，MockLLMCompletion 補上了串流支援（原本會直接 raise）。

## 後續若要接真實 LLM 驗證，只需 graphrag init 後把 indexing_method 改為 fact-leading 並填入模型金鑰即可。