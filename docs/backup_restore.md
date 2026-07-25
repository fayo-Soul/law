# 数据备份与恢复

## 概述

本文档说明如何备份和恢复法律咨询 RAG 系统的关键数据。

**需要备份的数据：**

| 数据类型 | 备份方式 | 重要性 |
|---|---|---|
| MySQL 问答库 | `mysqldump` | ⭐⭐⭐ |
| 法律条文原始文件 | 文件复制 | ⭐⭐⭐ |
| 案例原始数据 | 文件复制 | ⭐⭐⭐ |
| 结构化问答对 | 文件复制 | ⭐⭐ |
| 配置文件 (.env, config.ini) | 文件复制 | ⭐⭐ |
| 模型文件 | 手动管理 | ⭐⭐ |
| Milvus 向量库 | 数据卷/云服务 | ⭐⭐ |
| Redis 缓存 | 不备份 | ⭐ |

---

## 快速备份

使用备份脚本一键全量备份：

```bash
# 全量备份（MySQL + 数据文件）
python scripts/backup.py

# 仅备份 MySQL
python scripts/backup.py --type mysql

# 仅备份数据文件
python scripts/backup.py --type data

# 指定输出目录
python scripts/backup.py --output /data/backups
```

备份输出到 `backups/` 目录，包含：

```
backups/
├── mysql_law_qa_20260706_143000.sql   # MySQL dump
├── data_files_20260706_143000/         # 原始数据文件
│   ├── 法律条文/
│   ├── case_rag/data/raw/
│   ├── 法律问答.txt
│   ├── law_qa_pairs.jsonl
│   ├── .env.example
│   ├── config.ini
│   └── models_manifest.json
└── backup_report.json                  # 备份报告
```

---

## 详细说明

### 1. MySQL 备份

使用 `mysqldump` 导出问答库：

```bash
# 手动备份
mysqldump -h 127.0.0.1 -P 3306 -u root -p fzt_law > backup.sql

# 从 Docker 容器内备份
docker exec mysql-container mysqldump -u root -p fzt_law > backup.sql
```

**恢复：**

```bash
# 恢复到 MySQL
mysql -h 127.0.0.1 -P 3306 -u root -p fzt_law < backup.sql

# 从 Docker 恢复
cat backup.sql | docker exec -i mysql-container mysql -u root -p fzt_law
```

### 2. 原始数据文件

**需要备份的文件和目录：**

- `法律条文/` — 法律条文原始文档
- `case_rag/data/raw/` — 案例原始数据
- `法律问答.txt` — 结构化问答对源文件
- `law_qa_pairs.jsonl` — 解析后的问答对
- `.env` / `config.ini` — 配置文件（注意 `.env` 包含密钥，应加密存储）

**恢复：**

```bash
# 将备份的文件复制回项目根目录
cp -r backups/data_files_20260706_143000/法律条文/ .
cp -r backups/data_files_20260706_143000/case_rag/data/raw/ case_rag/data/raw/
cp backups/data_files_20260706_143000/法律问答.txt .
cp backups/data_files_20260706_143000/law_qa_pairs.jsonl .

# 恢复后重新导入向量库
python main.py setup-all
```

### 3. Milvus 向量库

Milvus 的备份方式取决于部署方式：

#### Docker 本地部署

备份 Milvus 数据卷：

```bash
# 找到 Milvus 数据卷位置
docker volume inspect milvus_data

# 备份数据卷目录
tar czf milvus_backup.tar.gz /var/lib/docker/volumes/milvus_data/
```

恢复时重新创建容器并挂载备份的数据卷。

#### 云服务（Zilliz Cloud / 阿里云向量检索等）

使用云服务商提供的备份功能：

- Zilliz Cloud：控制台 → 备份 → 创建备份
- 阿里云向量检索：控制台 → 备份恢复

### 4. 模型文件

模型文件通常较大，建议单独管理，不随代码备份。

**推荐做法：**

1. 模型文件存储在共享存储（NAS / 对象存储）
2. 通过 `models_manifest.json` 记录版本和路径
3. 部署时通过脚本自动下载对应版本

**恢复：**

```bash
# 根据 models_manifest.json 检查模型文件完整性
python scripts/check_services.py
```

### 5. 定时备份

#### Linux (crontab)

```bash
# 每天凌晨 2 点备份
0 2 * * * cd /path/to/project && python scripts/backup.py --output /data/backups/$(date +\%Y\%m\%d) >> /var/log/law_backup.log 2>&1

# 保留最近 30 天备份
0 3 * * * find /data/backups/ -maxdepth 1 -name "backup_report.json" -mtime +30 -exec rm -rf {} \;
```

#### Windows (任务计划程序)

创建 PowerShell 脚本 `scheduled_backup.ps1：

```powershell
cd E:\Files\项目\法律咨询RAG\law
$date = Get-Date -Format "yyyyMMdd"
$output = "E:\Backups\$date"
python scripts\backup.py --output $output
```

在"任务计划程序"中设置每天运行。

---

## 恢复后检查

恢复数据后，执行以下检查：

```bash
# 1. 检查配置
python -c "from base.config import Config; c=Config(); print('MYSQL_HOST:', c.MYSQL_HOST)"

# 2. 检查模型完整性
python scripts/check_services.py

# 3. 启动服务并检查健康
python api_server.py &
curl http://localhost:8000/api/v1/health

# 4. 测试问答
curl -X POST http://localhost:8000/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "劳动法加班费怎么算"}'
```

---

## 灾难恢复流程

如果服务器完全损坏，按以下步骤恢复：

1. **部署新环境**：参考 [deployment.md](deployment.md)
2. **恢复配置文件**：从安全的存储中取出 `.env` 和 `config.ini`
3. **恢复数据文件**：将备份的数据文件复制到项目目录
4. **恢复 MySQL**：导入 SQL dump
5. **恢复 Milvus**：恢复数据卷或重新导入
6. **重新导入向量库**：`python main.py setup-all`
7. **检查并启动**：运行 `check_services.py` 确认一切正常
