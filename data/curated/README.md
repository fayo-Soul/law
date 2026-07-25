# 精选法律知识库维护规范

## 定位

该目录保存经过人工核对、面向具体律师业务场景的少量高质量资料。正式问答默认使用
精选 Collection，不直接使用历史批量抓取数据。

## 每条记录的必填字段

- `document_id`：稳定、不可复用的业务 ID。
- `law_name`：法律法规完整名称。
- `article_number`：具体条号。
- `source`：法律领域。
- `document_type`：法规、司法解释或内部资料。
- `effective_status`：现行有效、已修订或已废止。
- `version`：该条款对应版本。
- `source_url`：国家机关或司法机关权威来源。
- `content`：核对后的条文原文。
- `business_summary`：律师使用时需要核对的事实与证据。
- `business_scenarios`：用于检索增强的业务关键词。

## 更新流程

1. 从国家法律法规数据库、全国人大、国务院、最高人民法院或主管部门核对原文。
2. 新增或修改 JSONL 记录，不直接修改 Milvus 数据。
3. 运行 `python scripts/ingest_curated_knowledge.py` 重建精选 Collection。
4. 运行 `python scripts/evaluate_retrieval.py`，检索门禁必须通过。
5. 运行生成式问答抽样评测并由法律人员复核。
6. 通过审核后再切换生产环境 Collection。

## 禁止事项

- 禁止从搜索摘要直接复制法条而不回到权威原文核对。
- 禁止把 AI 生成内容当作法条原文。
- 禁止覆盖旧版本而不记录版本和效力状态。
- 禁止为了让评测通过而在问题或答案中写入不可泛化的特殊标记。
