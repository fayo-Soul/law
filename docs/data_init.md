# 数据初始化指南

## 概述

系统启动前需要初始化三类数据：

1. **问答对** → MySQL `fzt_qa` 表
2. **法律条文** → Milvus `fzt_law.fzt_legal_articles` 集合
3. **案例文书** → Milvus `fzt_case.fzt_case_hybrid` 集合

## 一键初始化

```bash
python main.py setup-all
```

该命令依次执行 import-qa、ingest-articles、ingest-cases。

## 分步初始化

### 导入问答对

```bash
python main.py import-qa
```

从 `法律问答.txt` 解析结构化问答对，写入 MySQL `fzt_qa` 表。

**数据格式要求：**

```
问：公司拖欠工资怎么办？
答：根据《劳动法》第五十条，工资应当以货币形式按月支付...

问：离婚需要什么条件？
答：根据《民法典》第一千零七十九条...
```

### 导入法律条文

```bash
python main.py ingest-articles
```

将 `法律条文/` 目录下的文件切片后写入 Milvus。

支持格式：`.md` / `.txt`

文档会被分割为父子块（父块 512 字符，子块 128 字符），子块用于检索，父块用于生成上下文。

### 导入案例

```bash
python main.py ingest-cases
```

将 `case_rag/data/raw/` 目录下的案例文书写入 Milvus。

## 数据文件说明

| 文件 | 说明 |
|---|---|
| `法律问答.txt` | 法律问答对源文件 |
| `法律条文/` | 法律知识库（目录结构按法律部门组织） |
| `case_rag/data/raw/` | 案例文书库 |
| `bert_balanced.jsonl` | BERT 训练数据（清洗后） |
| `preprocessed_data.jsonl` | 预处理后的训练数据 |

## 验证数据是否导入成功

```bash
python -c "
from base.config import Config
from rag_qa.core.rag_system import RAGSystem
cfg = Config()
rag = RAGSystem(cfg)
# 测试检索
results = rag.retrieve('劳动法工作时间')
print(f'检索到 {len(results)} 条结果')
"
```
