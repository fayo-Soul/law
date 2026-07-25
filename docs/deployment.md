# 部署指南

## 环境依赖

| 组件 | 版本要求 | 说明 |
|---|---|---|
| Python | >= 3.10 | |
| MySQL | 5.7+ | 存储结构化问答对 |
| Redis | 6.x+ | 缓存（可选） |
| Milvus | 2.x | 向量数据库 |
| DashScope API | - | 通义千问 LLM 服务 |

## 部署方式

### 方式一：直接部署

```bash
# 1. 创建虚拟环境
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# 或
.venv\Scripts\activate      # Windows

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置环境变量
cp .env.example .env
# 编辑 .env 填入真实配置

# 4. 初始化数据
python main.py setup-all

# 5. 启动服务
python api_server.py
# 访问 http://localhost:8000
```

### 方式二：Docker 部署

要求：已安装 Docker 和 Docker Compose。

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env 填入真实配置

# 2. 启动服务
docker compose up -d

# 3. 查看日志
docker compose logs -f

# 4. 停止服务
docker compose down
```

**注意事项：**
- 镜像不包含大模型文件，通过 volume 挂载 `rag_qa/models/` 和 `bert_law_classifier/`
- Milvus 作为外部服务连接（需提前部署或使用现有实例）
- 首次启动后需执行数据初始化：`docker compose exec law-api python main.py setup-all`

## 环境变量说明

所有配置通过 `.env` 或环境变量注入。关键配置项：

| 变量 | 说明 | 默认值 |
|---|---|---|
| `MYSQL_HOST` | MySQL 地址 | 127.0.0.1 |
| `MYSQL_PORT` | MySQL 端口 | 3306 |
| `MYSQL_USER` | MySQL 用户名 | root |
| `MYSQL_PASSWORD` | MySQL 密码 | - |
| `MYSQL_DATABASE` | 数据库名 | fzt_law |
| `REDIS_HOST` | Redis 地址 | 127.0.0.1 |
| `REDIS_PORT` | Redis 端口 | 6379 |
| `MILVUS_HOST` | Milvus 地址 | 127.0.0.1 |
| `MILVUS_PORT` | Milvus 端口 | 19530 |
| `DASHSCOPE_API_KEY` | 通义千问 API Key | - |
| `LLM_MODEL` | 模型名称 | qwen-plus |

## 生产环境注意事项

1. **密钥管理**：使用环境变量注入，不要将 `.env` 提交到仓库
2. **CORS 配置**：生产环境限制 `allow_origins` 为具体域名
3. **日志轮转**：配置 logrotate 防止日志文件过大
4. **健康检查**：确保 `/health` 端点可正常访问
5. **资源监控**：监控 MySQL/Milvus/Redis 连接数
