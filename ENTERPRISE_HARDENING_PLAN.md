# 法律咨询 RAG 项目企业化改造与上线交付方案

> 适用对象：希望把当前法律咨询 RAG Demo 项目，继续打磨成"能交付、能部署、能上线、企业内部可用"的项目。  
> 当前项目根目录：`E:\Files\项目\法律咨询RAG\law`

---

## 0. 当前项目判断

当前项目已经具备一个法律问答系统的核心雏形：

- 有 FastAPI 后端入口：`api_server.py`
- 有命令行入口：`main.py`
- 有前端页面：`web/index.html`
- 有法律条文知识库：`法律条文/`
- 有案例库：`case_rag/data/raw/`
- 有结构化问答对：`法律问答.txt`、`law_qa_pairs.jsonl`
- 有 BERT 法律领域分类模型
- 有法律条文 RAG 检索
- 有案例 RAG 检索
- 有 MySQL、Redis、Milvus、DashScope/Qwen 调用链路

但是它目前更像"课程项目 / 原型 Demo / 本地实验项目"，距离企业可交付项目还有明显差距。

主要问题包括：

- 明文密钥写在 `config.ini`
- 中文内容出现乱码，影响维护
- 没有完善 README
- `pyproject.toml` 没有声明真实依赖
- 没有测试
- 没有 Docker 部署方案
- 没有生产环境配置规范
- 没有日志脱敏、错误码、异常处理体系
- 服务启动时强依赖 MySQL、Redis、Milvus、模型和外部 LLM，容错较弱
- 前端只是静态 Demo，缺少企业使用场景下的必要交互和错误提示
- **没有 RAG 质量评估体系，无法衡量检索和回答质量**
- **没有知识更新机制，法律法规变更后无法及时更新**
- **没有对话记忆，多轮咨询无法跟踪上下文**
- **没有 CI/CD，代码变更无法自动化验证**
- **没有个人信息保护合规措施**

本方案目标不是"重写项目"，而是在保留现有核心能力的基础上，分阶段把项目打磨成可以交付、可以演示、可以部署上线的企业可用项目。

---

## 1. 改造总目标

最终希望项目达到以下状态：

1. 可以交付给别人安装运行。
2. 可以用 Docker 或一键脚本启动。
3. 配置、密钥、数据库地址不会写死在代码里。
4. 有清晰 README，别人能看懂怎么部署、怎么初始化数据、怎么使用。
5. 有基本测试，避免每次改代码都把主流程改坏。
6. 服务出错时不会直接崩溃，能返回友好错误。
7. 日志能帮助排查问题，但不会泄露密钥和隐私。
8. API 接口结构清晰，前端调用稳定。
9. 法律咨询回答有免责声明，降低企业合规风险。
10. 项目目录清晰，后续继续开发不容易乱。
11. **RAG 检索质量可评估，幻觉率可追踪。**
12. **知识库可增量更新，支持法律条文和案例的新增、废止、修订。**
13. **对话可延续，多轮咨询保持上下文。**
14. **有自动化 CI/CD，提交代码后自动测试和构建。**
15. **符合个人信息保护法（PIPL）基本要求，用户数据可管理。**

---

## 2. 推荐改造阶段

建议分成 8 个阶段做，不要一口气全改。

| 阶段 | 目标 | 优先级 |
|---|---|---|
| 第 1 阶段 | 先让项目"可读、可配置、可安全管理密钥" | 必做 |
| 第 2 阶段 | 修复编码乱码和文档缺失问题 | 必做 |
| 第 3 阶段 | 重构配置和启动流程 | 必做 |
| 第 4 阶段 | 增强 API 稳定性和错误处理 | 必做 |
| 第 5 阶段 | 增加测试、CI/CD 和质量检查 | 必做 |
| 第 6 阶段 | 增加 Docker 和部署文档 | 必做 |
| 第 7 阶段 | 优化前端和用户体验 | 建议 |
| 第 8 阶段 | 企业级安全、审计、权限、监控 | 上线前建议 |

---

# 第 1 阶段：密钥与配置安全改造

## 1.1 为什么要改

当前 `config.ini` 里面包含：

- MySQL 地址、账号、密码
- Redis 地址、密码
- Milvus 地址
- DashScope API Key
- 客服电话

这在企业项目中是不合格的。

原因：

1. 密钥一旦提交给别人，就可能泄露。
2. 不同环境的配置不同，比如本地、测试服务器、生产服务器。
3. 如果以后上传 GitHub，API Key 可能被扫描出来并盗用。
4. 企业部署时通常不允许把密码写死在代码仓库里。

## 1.2 怎么改

### 第一步：新增 `.env.example`

创建一个示例环境变量文件，只放配置名，不放真实密码。

建议新增文件：

```text
.env.example
```

内容示例：

```env
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=change_me
MYSQL_DATABASE=fzt_law
MYSQL_LAW_QA_TABLE=fzt_qa

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=change_me
REDIS_DB=1

MILVUS_HOST=127.0.0.1
MILVUS_PORT=19530
MILVUS_DATABASE_NAME=fzt_law
MILVUS_COLLECTION_NAME=fzt_legal_articles

CASE_MILVUS_DATABASE_NAME=fzt_case
CASE_MILVUS_COLLECTION_NAME=fzt_case_hybrid

LLM_MODEL=qwen-plus
DASHSCOPE_API_KEY=change_me
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

CUSTOMER_SERVICE_PHONE=13575008699
LOG_FILE=logs/app.log
```

### 第二步：新增 `.env`

本地运行时复制一份：

```powershell
Copy-Item .env.example .env
```

然后把 `.env` 里的 `change_me` 改成真实配置。

注意：`.env` 不能提交到代码仓库。

### 第三步：修改 `.gitignore`

确保 `.gitignore` 里包含：

```gitignore
.env
*.log
logs/
__pycache__/
.venv/
```

如果以后项目初始化 Git 仓库，`.env` 就不会被提交。

### 第四步：修改 `base/config.py`

当前 `base/config.py` 只读 `config.ini`。

建议改成：

1. 优先读取环境变量。
2. 如果环境变量没有，再读取 `config.ini`。
3. 如果都没有，再使用默认值。

示例思路：

```python
import os
import configparser

def get_config(config, section, key, env_name, fallback=None):
    return os.getenv(env_name) or config.get(section, key, fallback=fallback)
```

然后：

```python
self.MYSQL_HOST = get_config(self.config, "mysql", "host", "MYSQL_HOST", "127.0.0.1")
self.DASHSCOPE_API_KEY = get_config(self.config, "llm", "dashscope_api_key", "DASHSCOPE_API_KEY", "")
```

### 第五步：安装 `python-dotenv`

在 `requirements.txt` 中加入：

```text
python-dotenv
```

然后在 `base/config.py` 开头加载 `.env`：

```python
from dotenv import load_dotenv
load_dotenv()
```

## 1.3 修改后的成果

完成后项目会变成：

- 代码仓库里不再保存真实密钥。
- 企业部署时可以通过环境变量注入配置。
- 本地开发人员可以各自维护自己的 `.env`。
- 后续 Docker、服务器部署都会更容易。

---

# 第 2 阶段：修复中文乱码和文档缺失

## 2.1 为什么要改

当前很多文件在终端里显示为乱码，例如：

- `config.ini`
- `main.py`
- `api_server.py`
- `web/index.html`
- `rag_qa/core/*.py`

虽然 Python 可能还能运行，但对后续维护非常不友好。

企业项目里，代码和文档必须可读。

否则会出现：

- 新人看不懂代码
- 前端页面显示乱码
- 日志难以排查
- prompt 模板不可维护
- 法律回答里的免责声明可能显示异常

## 2.2 怎么改

### 第一步：确认文件真实编码

用编辑器打开文件，确认它们到底是：

- UTF-8
- GBK
- UTF-8 with BOM
- 其他编码

如果你用 PyCharm：

1. 打开文件。
2. 看右下角编码。
3. 如果不是 UTF-8，选择转换为 UTF-8。

如果你用 VS Code：

1. 右下角点击编码。
2. 选择 `Reopen with Encoding`。
3. 尝试 GBK 或 UTF-8。
4. 正确显示中文后，选择 `Save with Encoding -> UTF-8`。

### 第二步：优先修复这些文件

先修最关键文件：

```text
config.ini
main.py
api_server.py
web/index.html
base/config.py
rag_qa/core/prompts.py
rag_qa/core/law_query_classifier.py
rag_qa/core/document_processor.py
rag_qa/core/rag_system.py
case_rag/searcher.py
```

### 第三步：重新写清楚注释

不要只是把乱码转回来，也建议顺手整理注释。

例如把：

```python
# 鍒濆鍖栨硶寰嬪姪鎵?
```

改成：

```python
# 初始化法律问答助手，启动时会加载数据库连接、向量库和本地模型
```

### 第四步：修复前端页面文案

`web/index.html` 中有大量乱码，建议替换为正常中文。

例如：

```html
<title>法智通 · 智能法律问答</title>
```

页面中建议保留这些文案：

- 智能法律咨询
- 新建对话
- 最近对话
- 答案仅供参考，具体案件请咨询专业律师
- 请输入您的法律问题

### 第五步：新增 `README.md`

当前 `pyproject.toml` 写了：

```toml
readme = "README.md"
```

但是项目根目录没有看到有效 README。

建议创建：

```text
README.md
```

内容至少包括：

```md
# 法律咨询 RAG 系统

## 项目简介

这是一个基于法律条文、案例库、结构化问答对和大语言模型的智能法律问答系统。

## 核心能力

- 法律问题分类
- 结构化问答检索
- 法律条文 RAG 检索
- 法律案例检索
- 大模型生成回答
- Web 聊天页面
- FastAPI 接口服务

## 技术栈

- Python 3.10+
- FastAPI
- MySQL
- Redis
- Milvus
- Transformers
- Sentence Transformers
- DashScope / Qwen

## 快速启动

1. 创建虚拟环境
2. 安装依赖
3. 配置 `.env`
4. 初始化数据
5. 启动 API 服务

## 注意事项

本系统回答仅供参考，不构成正式法律意见。
```

## 2.3 修改后的成果

完成后：

- 代码可读性明显提升。
- 前端页面正常显示中文。
- 日志和错误信息能看懂。
- 新人可以通过 README 理解项目。
- 后续维护成本大幅降低。

---

# 第 3 阶段：整理项目目录结构

## 3.1 为什么要改

当前项目根目录里混合了：

- 源代码
- 大模型权重
- 数据集
- 清洗后的训练数据
- 前端页面
- 日志
- 脚本
- 配置文件

短期能跑，但长期维护会乱。

企业项目一般会区分：

- 应用代码
- 配置
- 数据
- 脚本
- 文档
- 测试
- 部署文件

## 3.2 推荐目录结构

建议逐步整理成：

```text
law/
  app/
    api/
      routes.py
      schemas.py
    core/
      assistant.py
      llm.py
      settings.py
    services/
      qa_service.py
      article_rag_service.py
      case_rag_service.py
    main.py

  base/
  rag_qa/
  mysql_qa/
  case_rag/

  web/
    index.html

  scripts/
    train_bert.py
    import_qa.py
    ingest_articles.py
    ingest_cases.py

  docs/
    deployment.md
    api.md
    data_init.md
    faq.md

  tests/
    test_config.py
    test_api_health.py
    test_prompt.py

  data/
    raw/
    processed/

  docker/
    Dockerfile
    docker-compose.yml

  .env.example
  .gitignore
  README.md
  requirements.txt
```

不建议第一天就大规模搬文件。先新增 `docs/`、`tests/`、`docker/`，然后慢慢把脚本和文档整理进去。

## 3.3 怎么改

### 第一步：新增文档目录

```text
docs/
```

建议放：

```text
docs/deployment.md
docs/data_init.md
docs/api.md
docs/ops.md
```

### 第二步：新增测试目录

```text
tests/
```

先不要追求完整测试，先写最小测试。

### 第三步：保留现有模块

当前这些目录先不动：

```text
base/
rag_qa/
mysql_qa/
case_rag/
```

因为它们已经承载核心逻辑，大规模移动容易引入路径错误。

### 第四步：后续逐步抽象 `LawAssistant`

当前 `LawAssistant` 在 `main.py` 中。

建议后续拆到：

```text
app/core/assistant.py
```

这样 `main.py` 和 `api_server.py` 都可以复用。

## 3.4 修改后的成果

完成后：

- 项目结构更像正式工程。
- 文档、测试、部署文件有明确位置。
- 后续新人不会不知道文件该放哪里。
- 以后可以逐步把 `main.py` 拆小。

---

# 第 4 阶段：优化启动流程和依赖管理

## 4.1 为什么要改

当前 `api_server.py` 在模块加载时就执行：

```python
assistant = LawAssistant()
```

这意味着只要启动 API，就会立刻初始化：

- MySQL
- Redis
- Milvus
- BERT 分类器
- BGE-M3 模型
- BGE reranker
- 案例检索器
- DashScope client

如果其中任何一个失败，服务可能直接启动失败。

企业项目中，服务应该：

- 尽可能启动成功
- 明确检查依赖状态
- 对失败依赖给出清晰错误
- 不要在导入模块时做太重的事情

## 4.2 怎么改

### 第一步：把初始化放到 FastAPI 生命周期里

不要在文件顶层直接初始化。

建议改成：

```python
from contextlib import asynccontextmanager

assistant = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global assistant
    app.state.ready = False
    try:
        assistant = LawAssistant()
        app.state.ready = True
    except Exception as e:
        logger.error(f"LawAssistant 初始化失败: {e}")
    yield
    # 如果有连接需要关闭，可以在这里关闭

app = FastAPI(
    title="法律问答系统",
    version="1.0.0",
    lifespan=lifespan
)
```

### 第二步：新增启动状态

增加一个状态变量：

```python
app.state.ready = False
```

初始化成功后：

```python
app.state.ready = True
```

### 第三步：改造健康检查接口

当前：

```python
@app.get("/api/health")
def health():
    return {"status": "ok"}
```

建议改成：

```python
@app.get("/api/health")
def health():
    return {
        "status": "ok",
        "ready": app.state.ready
    }
```

再新增：

```python
@app.get("/api/ready")
def ready():
    if not app.state.ready:
        return JSONResponse(status_code=503, content={"status": "starting"})
    return {"status": "ready"}
```

### 第四步：新增依赖检查脚本

新增：

```text
scripts/check_services.py
```

检查：

- MySQL 是否可连接
- Redis 是否可连接
- Milvus 是否可连接
- DashScope Key 是否配置
- 模型目录是否存在

运行方式：

```powershell
python scripts/check_services.py
```

## 4.3 修改后的成果

完成后：

- 服务启动逻辑更规范。
- 健康检查可以区分"进程活着"和"系统真的可用"。
- 上线部署时可以用 `/api/ready` 判断是否可以接流量。
- 排查环境问题更容易。

---

# 第 5 阶段：API 接口规范化

## 5.1 为什么要改

当前 `/api/chat` 返回：

```python
return {"answer": answer}
```

但是响应模型里定义了：

```python
class ChatResponse(BaseModel):
    answer: str
    source_filter: str | None = None
```

前端又试图读取：

```javascript
data.source_filter
```

这说明接口设计还不够严谨。

企业项目需要接口稳定，否则前端和后端容易互相踩坑。

## 5.2 怎么改

### 第一步：明确请求结构

建议：

```python
class ChatRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=2000)
    source_filter: str | None = None
    session_id: str | None = None
```

### 第二步：明确响应结构

建议：

```python
class ReferenceItem(BaseModel):
    type: str  # "article" | "case" | "qa"
    title: str
    source: str
    content: str
    score: float | None = None

class ChatResponse(BaseModel):
    answer: str
    source_filter: str | None = None
    request_id: str
    used_rag: bool = False
    used_fallback: bool = False
    references: list[ReferenceItem] = []
    disclaimer: str = "以上内容由 AI 生成，仅供参考，不构成正式法律意见。"
```

字段解释：

- `answer`：最终回答
- `source_filter`：分类得到的法律领域
- `request_id`：请求编号，排查日志用
- `used_rag`：是否使用了 RAG 检索
- `used_fallback`：是否发生了降级（某个环节失败走了兜底）
- `references`：引用的法律条文或案例
- `disclaimer`：免责声明

### 第三步：为错误返回统一格式

建议错误返回：

```json
{
  "code": "SERVICE_UNAVAILABLE",
  "message": "法律问答服务暂时不可用，请稍后重试",
  "request_id": "xxx"
}
```

常见错误码：

| 错误码 | HTTP 状态码 | 含义 |
|---|---|---|
| INVALID_REQUEST | 400 | 请求参数错误 |
| SERVICE_NOT_READY | 503 | 服务尚未初始化完成 |
| RATE_LIMITED | 429 | 请求过于频繁 |
| LLM_ERROR | 502 | 大模型调用失败 |
| RETRIEVAL_ERROR | 502 | 检索失败 |
| INTERNAL_ERROR | 500 | 系统内部错误 |

### 第四步：给问题长度加限制

防止用户输入过长导致成本或性能问题。

例如：

```python
from pydantic import BaseModel, Field

class ChatRequest(BaseModel):
    question: str = Field(..., min_length=1, max_length=2000)
```

### 第五步：增加请求 ID

在接口里生成：

```python
import uuid
request_id = str(uuid.uuid4())
```

日志里也带上这个 ID。

### 第六步：API 版本化

建议将路由加上版本前缀，便于后续演进：

```text
POST /api/v1/chat
GET  /api/v1/health
GET  /api/v1/ready
```

旧路由通过重定向保持兼容。

## 5.3 修改后的成果

完成后：

- 前后端字段一致。
- 线上出问题可以通过 `request_id` 查日志。
- 错误信息更友好。
- 接口可以长期稳定演进。

---

# 第 6 阶段：日志、异常处理和脱敏

## 6.1 为什么要改

法律咨询项目可能涉及用户隐私。

用户可能输入：

- 身份证号
- 手机号
- 姓名
- 公司名称
- 案件细节
- 合同信息

如果日志直接记录全部内容，会有合规风险。

同时，API Key、数据库密码也不能出现在日志里。

## 6.2 怎么改

### 第一步：统一日志格式

建议日志包含：

```text
时间 | 等级 | 模块 | request_id | 消息
```

示例：

```text
2026-07-06 10:00:00 | INFO | LawAssistant | request_id=xxx | 收到法律问题
```

### 第二步：新增脱敏函数

新增：

```text
base/security.py
```

示例：

```python
import re

def mask_phone(text: str) -> str:
    return re.sub(r"1[3-9]\d{9}", lambda m: m.group(0)[:3] + "****" + m.group(0)[-4:], text)

def mask_id_card(text: str) -> str:
    return re.sub(r"\d{17}[\dXx]", lambda m: m.group(0)[:6] + "********" + m.group(0)[-4:], text)

def mask_api_key(text: str) -> str:
    return re.sub(r"(sk-)[a-f0-9]{32}", lambda m: m.group(1) + "****" + m.group(0)[-4:], text)

def mask_sensitive_text(text: str) -> str:
    text = mask_phone(text)
    text = mask_id_card(text)
    text = mask_api_key(text)
    return text
```

### 第三步：记录用户问题时先脱敏

当前类似：

```python
logger.info(f"收到问题：{query}")
```

建议改成：

```python
logger.info(f"收到问题：{mask_sensitive_text(query)}")
```

### 第四步：异常不要直接返回内部细节

当前有些地方会：

```python
return f"处理问题时出错：{e}"
```

生产环境不建议把 `e` 原样返回给用户。

建议：

```python
logger.exception("RAG 处理失败")
return "抱歉，系统暂时无法完成检索，请稍后重试或联系人工客服。"
```

## 6.3 修改后的成果

完成后：

- 日志能排查问题。
- 日志不会轻易泄露用户隐私。
- 用户看到的是友好错误，不是 Python 报错。
- 更符合企业合规要求。

---

# 第 7 阶段：RAG 检索链路增强

## 7.1 为什么要改

当前系统的 RAG 链路已经存在：

1. BERT 分类
2. 法律条文检索
3. 案例检索
4. 拼接上下文
5. 大模型回答

但是企业可用项目还需要：

- 明确引用来源
- 防止模型胡说
- 检索不到时给出合理兜底
- 分类错误时有 fallback
- 检索结果可解释

## 7.2 怎么改

### 第一步：返回引用来源

当前 `format_articles()` 只拼接内容，没有把文件名、法律名称、条文信息返回给前端。

建议在向量库文档 metadata 中保留：

```python
metadata = {
    "source": "民法",
    "file_path": "...",
    "law_name": "民法典",
    "timestamp": "..."
}
```

接口响应中加入：

```json
"references": [
  {
    "type": "article",
    "title": "民法典",
    "source": "民法",
    "content": "..."
  },
  {
    "type": "case",
    "title": "某某合同纠纷案",
    "case_number": "..."
  }
]
```

### 第二步：修改 Prompt，要求基于材料回答

在 `rag_qa/core/prompts.py` 中的法律问答 prompt 建议明确：

```text
你是法律咨询助手。
请优先依据【相关法律条文】和【相关参考案例】回答。
如果材料不足，请明确说明"根据当前资料无法确定"，不要编造法条、案号或判决结果。
回答仅供参考，不构成正式法律意见。
```

### 第三步：检索不到时不要硬编

当前如果条文和案例都没检索到，仍然会让 LLM 回答。

建议区分：

```python
if not article_docs and not cases:
    return "未检索到足够相关的法律条文或案例。建议补充更多事实信息，或咨询专业律师。"
```

也可以保留 LLM 兜底，但必须提示：

```text
以下回答未基于本地法律知识库检索结果，仅供一般参考。
```

### 第四步：领域过滤失败时自动全库检索

当前已经有类似逻辑：

```python
if not article_docs:
    article_docs = self.rag_system.retrieve_and_merge(query, source_filter=None)
```

建议保留，并在响应里记录：

```json
"used_fallback": true
```

### 第五步：检索结果加分数

检索结果建议返回相似度分数：

```python
{
    "content": "...",
    "score": 0.82
}
```

这样后续可以判断结果质量。

## 7.3 修改后的成果

完成后：

- 回答更可信。
- 用户可以看到依据来源。
- 模型不容易编造。
- 企业演示时更有说服力。

---

# 第 8 阶段：结构化问答库增强

## 8.1 为什么要改

当前结构化问答库使用：

- MySQL 存问答
- Redis 缓存
- BM25 检索相似问题

这个设计是合理的。

但是企业项目中，问答库需要更可管理：

- 支持新增、修改、删除问答
- 支持领域标签
- 支持审核状态
- 支持命中统计
- 支持低置信度 fallback

## 8.2 怎么改

### 第一步：扩展 MySQL 表结构

当前表：

```sql
id
question
answer
domain
create_time
```

建议扩展为：

```sql
CREATE TABLE IF NOT EXISTS fzt_qa (
    id INT AUTO_INCREMENT PRIMARY KEY,
    question TEXT NOT NULL,
    answer TEXT NOT NULL,
    domain VARCHAR(50) DEFAULT '',
    status VARCHAR(20) DEFAULT 'enabled',
    hit_count INT DEFAULT 0,
    created_by VARCHAR(50) DEFAULT '',
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 第二步：查询时只查启用状态

```sql
SELECT question FROM fzt_qa WHERE status='enabled'
```

### 第三步：命中后增加命中次数

```sql
UPDATE fzt_qa SET hit_count = hit_count + 1 WHERE id = %s
```

### 第四步：增加管理接口

后续可以加：

```text
GET /api/v1/admin/qa
POST /api/v1/admin/qa
PUT /api/v1/admin/qa/{id}
DELETE /api/v1/admin/qa/{id}
```

如果暂时不做后台，也可以先做导入脚本。

## 8.3 修改后的成果

完成后：

- 企业可以维护自己的高频问答。
- 常见问题命中更快。
- 可以统计用户关注点。
- 问答库有运营价值。

---

# 第 9 阶段：测试体系建设

## 9.1 为什么要改

现在项目没有测试。

这意味着：

- 改一个函数，不知道有没有影响主流程。
- 修一个 bug，可能引入另一个 bug。
- 上线前只能靠手动点击测试。
- 企业交付时不够专业。

## 9.2 怎么改

### 第一步：安装 pytest

在 `requirements.txt` 加：

```text
pytest
pytest-cov
httpx
```

### 第二步：新增测试目录

```text
tests/
```

建议按类型细分：

```text
tests/
  unit/              # 单元测试（不依赖外部服务）
  integration/       # 集成测试（可 mock 外部服务）
```

### 第三步：写配置读取测试

新增：

```text
tests/unit/test_config.py
```

测试目标：

- 可以读取默认配置
- 可以读取环境变量
- 必要字段不为空

示例：

```python
from base import Config

def test_config_loads():
    conf = Config()
    assert conf.MILVUS_PORT
    assert conf.LLM_MODEL
```

### 第四步：写健康检查测试

新增：

```text
tests/integration/test_api_health.py
```

示例：

```python
from fastapi.testclient import TestClient
from api_server import app

client = TestClient(app)

def test_health():
    response = client.get("/api/health")
    assert response.status_code == 200
    assert "status" in response.json()
```

注意：如果 `api_server.py` 启动时会初始化重型模型，这个测试会很慢。因此第 4 阶段的启动流程改造很重要。

### 第五步：写纯函数测试

适合测试的函数：

- `clean_bert_text`
- `remove_adjacent_duplicates`
- `valid_source_list`
- 敏感信息脱敏函数

例如：

```python
from base.security import mask_phone

def test_mask_phone():
    assert mask_phone("13812345678") == "138****5678"
```

### 第六步：增加 smoke test

新增：

```text
scripts/smoke_test.py
```

目标：

- 检查服务是否启动
- 请求 `/api/health`
- 可选：发一个简单问题到 `/api/chat`

## 9.3 修改后的成果

完成后：

- 每次修改后可以运行 `pytest`。
- 基础功能坏了能及时发现。
- 企业交付时更可信。
- 后续协作开发更安全。

---

# 第 10 阶段：Docker 化部署

## 10.1 为什么要改

企业上线时，不应该要求别人手动安装一堆依赖。

当前项目依赖很多：

- FastAPI
- Transformers
- PyTorch
- Milvus client
- LangChain
- Sentence Transformers
- MySQL client
- Redis client
- OpenAI SDK

如果每台服务器手动安装，容易出错。

Docker 可以把运行环境固定下来。

## 10.2 怎么改

### 第一步：新增 `Dockerfile`

建议新增：

```text
Dockerfile
```

示例：

```dockerfile
FROM python:3.10-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "api_server:app", "--host", "0.0.0.0", "--port", "8000"]
```

注意：当前项目带很多大模型文件，镜像可能非常大。企业部署时可以选择：

1. 模型打进镜像，部署简单但镜像巨大。
2. 模型挂载到容器，镜像小但部署稍复杂。

建议企业环境用第二种。

### 第二步：新增 `docker-compose.yml`

建议包括：

- API 服务
- Redis
- MySQL
- Milvus

Milvus 的 docker-compose 比较复杂，可以先把 Milvus 作为外部服务使用。

简单版本：

```yaml
services:
  law-api:
    build: .
    container_name: law-api
    ports:
      - "8000:8000"
    env_file:
      - .env
    volumes:
      - ./logs:/app/logs
      - ./rag_qa/models:/app/rag_qa/models
    restart: unless-stopped
```

### 第三步：新增部署文档

创建：

```text
docs/deployment.md
```

写清楚：

```md
## Docker 部署

1. 复制环境变量文件
2. 修改 `.env`
3. 构建镜像
4. 启动服务
5. 检查健康接口
```

命令：

```powershell
docker compose up -d --build
docker compose logs -f law-api
```

## 10.3 修改后的成果

完成后：

- 项目可以用 Docker 启动。
- 部署过程更稳定。
- 交付给别人更方便。
- 企业服务器迁移更容易。

---

# 第 11 阶段：数据初始化流程标准化

## 11.1 为什么要改

当前数据初始化分散在 `main.py` 子命令中：

```powershell
python main.py import-qa
python main.py ingest-articles
python main.py ingest-cases
python main.py setup-all
```

这是可以用的，但需要更清晰的文档和校验。

## 11.2 怎么改

### 第一步：写清楚初始化顺序

创建：

```text
docs/data_init.md
```

写：

```md
# 数据初始化说明

## 初始化顺序

1. 准备 MySQL
2. 准备 Redis
3. 准备 Milvus
4. 导入结构化问答对
5. 导入法律条文向量库
6. 导入案例向量库
7. 启动 API 服务
```

### 第二步：拆分独立脚本

虽然 `main.py` 有子命令，但企业项目建议脚本更直观。

建议新增：

```text
scripts/import_qa.py
scripts/ingest_articles.py
scripts/ingest_cases.py
```

它们内部调用已有函数即可。

例如：

```python
from main import parse_legal_qa, import_qa_to_mysql

if __name__ == "__main__":
    jsonl_path = parse_legal_qa()
    import_qa_to_mysql(jsonl_path)
```

### 第三步：初始化后输出统计

每个初始化脚本执行完成后输出：

- 成功导入多少问答
- 成功导入多少法律条文 chunk
- 成功导入多少案例 chunk
- 当前 Milvus collection 名称
- 当前数据库名称

### 第四步：支持不删除已有数据

当前有 `--no-drop`，要在文档里写清楚：

```powershell
python main.py ingest-articles --no-drop
python main.py ingest-cases --no-drop
```

解释：

- 默认会重建向量集合。
- `--no-drop` 表示保留旧数据。

## 11.3 修改后的成果

完成后：

- 数据初始化流程可复制。
- 部署人员知道先做什么、后做什么。
- 数据导入是否成功有明确反馈。
- 项目更像可交付系统。

---

# 第 12 阶段：前端页面增强

## 12.1 为什么要改

当前 `web/index.html` 是一个静态聊天页面，作为 Demo 可以，但企业使用还不够。

需要补充：

- 错误提示
- 加载状态
- 引用来源展示
- 法律免责声明
- 移动端适配
- 历史对话持久化
- 输入长度限制

## 12.2 怎么改

### 第一步：修复乱码

这是前端第一优先级。

页面上所有乱码都要替换为正常中文。

### 第二步：展示引用来源

当 API 返回：

```json
"references": []
```

前端显示：

```text
参考依据：
1. 民法典 第 xxx 条
2. 某某合同纠纷案
```

### 第三步：显示领域分类

如果返回：

```json
"source_filter": "民法"
```

前端显示：

```text
分类：民法
```

### 第四步：增加免责声明

建议在页面底部固定显示：

```text
本系统回答由 AI 生成，仅供参考，不构成正式法律意见。具体案件请咨询专业律师。
```

### 第五步：输入框限制

限制最多 2000 字。

前端：

```html
<textarea maxlength="2000"></textarea>
```

并显示：

```text
0 / 2000
```

### 第六步：错误提示更友好

如果请求失败，不要只显示 HTTP 错误。

建议：

```text
服务暂时不可用，请稍后重试。
如果多次失败，请联系管理员检查后端服务、数据库和向量库状态。
```

## 12.3 修改后的成果

完成后：

- 页面能正常给企业人员演示。
- 用户知道回答来自哪里。
- 用户知道答案不是正式法律意见。
- 出错时不至于困惑。

---

# 第 13 阶段：权限与管理后台

## 13.1 为什么要改

如果企业内部使用，至少要考虑：

- 谁能访问系统
- 谁能维护问答库
- 谁能查看日志
- 谁能导入数据

当前系统没有权限控制。

## 13.2 怎么改

### 第一步：API 增加简单 Token

先做最小版本。

`.env` 增加：

```env
API_TOKEN=change_me
```

请求时带：

```http
Authorization: Bearer your_token
```

后端校验：

```python
from fastapi import Header, HTTPException

def verify_token(authorization: str | None = Header(default=None)):
    if authorization != f"Bearer {Config().API_TOKEN}":
        raise HTTPException(status_code=401, detail="Unauthorized")
```

### 第二步：管理接口单独保护

普通聊天接口可以按企业需求决定是否开放。

管理接口必须保护：

```text
/api/admin/*
```

### 第三步：后续接企业登录系统

如果企业有统一登录，可以接：

- LDAP
- OAuth2
- 企业微信
- 飞书
- 钉钉

## 13.3 修改后的成果

完成后：

- 系统不再裸奔。
- 管理接口不会被随便访问。
- 后续可扩展企业 SSO。

---

# 第 14 阶段：监控和运维

## 14.1 为什么要改

上线后，不能只靠用户反馈"系统坏了"。

企业项目需要知道：

- 服务是否存活
- 请求量是多少
- 平均响应时间是多少
- 错误率是多少
- LLM 调用是否失败
- Milvus 检索是否失败
- MySQL/Redis 是否异常

## 14.2 怎么改

### 第一步：增加访问日志

每个请求记录：

- request_id
- path
- status_code
- cost_ms
- source_filter
- used_rag

### 第二步：增加耗时统计

在 `LawAssistant.answer()` 中记录：

- 分类耗时
- MySQL/Redis 检索耗时
- 法律条文检索耗时
- 案例检索耗时
- LLM 生成耗时

### 第三步：增加 Prometheus 指标

后续可以接：

```text
prometheus-fastapi-instrumentator
```

暴露：

```text
/metrics
```

### 第四步：部署监控面板

企业环境可使用：

- Prometheus
- Grafana
- Loki
- ELK

## 14.3 修改后的成果

完成后：

- 线上问题能定位。
- 可以看到系统性能瓶颈。
- 可以评估 LLM 成本和调用稳定性。
- 更符合企业运维要求。

---

# 第 15 阶段：法律合规与风险控制

## 15.1 为什么要改

法律咨询系统有特殊风险。

用户可能把系统回答当成正式法律意见。

企业上线时必须明确：

- 系统不是律师
- 回答仅供参考
- 不保证适用于所有案件
- 复杂案件建议咨询专业律师
- 不应该生成虚假法条
- 不应该承诺诉讼结果

## 15.2 怎么改

### 第一步：Prompt 中加入合规约束

在系统 prompt 中加入：

```text
你不是执业律师，不能替代正式法律服务。
回答应当谨慎，不能承诺案件结果。
如果事实不充分，应提示用户补充信息。
如果问题涉及重大权益，应建议咨询专业律师。
```

### 第二步：前端展示免责声明

页面底部展示：

```text
本系统回答由 AI 生成，仅供参考，不构成正式法律意见。
```

### 第三步：回答末尾增加提示

每次回答末尾可以加：

```text
以上内容仅供参考，具体处理建议结合完整证据材料咨询专业律师。
```

### 第四步：敏感问题触发人工提示

如果问题包含：

- 自杀
- 暴力
- 重大刑事案件
- 涉未成年人伤害
- 大额财产纠纷

建议提示人工介入。

## 15.3 修改后的成果

完成后：

- 降低法律风险。
- 回答更谨慎。
- 企业上线更容易通过内部审查。

---

# 第 16 阶段：模型和向量库管理

## 16.1 为什么要改

当前模型文件直接放在项目目录中：

```text
rag_qa/models/
bert_law_classifier/
```

这在本地演示可以，但企业部署时要考虑：

- 模型文件很大
- 镜像会很大
- 模型版本需要管理
- 多台服务器要保持一致

## 16.2 怎么改

### 第一步：模型路径配置化

`.env` 增加：

```env
MODEL_DIR=rag_qa/models
BERT_CLASSIFIER_DIR=rag_qa/models/bert_law_classifier
BGE_M3_DIR=rag_qa/models/bge-m3
BGE_RERANKER_DIR=rag_qa/models/bge-reranker-large
```

### 第二步：代码读取配置路径

不要在代码中写死：

```python
os.path.join(rag_qa_path, "models", "bge-m3")
```

改为：

```python
Config().BGE_M3_DIR
```

### 第三步：记录模型版本

新增：

```text
models_manifest.json
```

内容：

```json
{
  "bert_law_classifier": {
    "version": "2026-06-27",
    "path": "rag_qa/models/bert_law_classifier"
  },
  "bge_m3": {
    "version": "local",
    "path": "rag_qa/models/bge-m3"
  }
}
```

### 第四步：部署时挂载模型目录

Docker Compose 中：

```yaml
volumes:
  - ./rag_qa/models:/app/rag_qa/models
```

## 16.3 修改后的成果

完成后：

- 模型路径可配置。
- 模型升级有记录。
- Docker 镜像不会被模型文件拖得过大。
- 多环境部署更灵活。

---

# 第 17 阶段：数据库和向量库备份

## 17.1 为什么要改

企业上线后，数据是资产。

需要备份：

- MySQL 问答库
- Redis 缓存可以不备份
- Milvus 向量库
- 原始法律条文
- 原始案例
- 模型文件
- 配置文件模板

## 17.2 怎么改

### 第一步：MySQL 定期备份

使用：

```powershell
mysqldump -h 主机 -u 用户 -p 数据库名 > backup.sql
```

### 第二步：原始数据备份

备份：

```text
法律条文/
case_rag/data/raw/
问答对/
```

### 第三步：Milvus 备份

Milvus 的备份根据部署方式不同而不同。

如果是 Docker 本地部署，需要备份：

- Milvus 数据卷
- etcd 数据
- minio 数据

如果是云服务，使用云服务提供的备份方案。

### 第四步：写恢复文档

创建：

```text
docs/backup_restore.md
```

包含：

- 如何备份
- 如何恢复
- 恢复后如何检查

## 17.3 修改后的成果

完成后：

- 数据损坏后可以恢复。
- 服务器迁移更安全。
- 企业交付更加完整。

---

# 第 18 阶段：RAG 质量评估体系（新增）

## 18.1 为什么要改

当前系统无法回答以下问题：

- 检索命中率是多少？
- 回答的准确率是多少？
- 法条幻觉率是多少？
- 修改了 RAG 链路后，效果是变好了还是变差了？

没有评估体系，就无法验收 RAG 系统的质量。对于法律咨询系统，错误的法条引用可能导致严重的法律后果，因此评估和监控回答质量是上线前的必要环节。

## 18.2 怎么改

### 第一步：构建标准评估数据集

新增：

```text
data/eval/qa_pairs.jsonl
```

每条数据包含：

```jsonl
{"question": "离婚后孩子的抚养权怎么判？", "expected_domain": "民法", "golden_docs": ["民法典 第一千零八十四条", "民法典 第一千零八十五条"]}
{"question": "公司拖欠工资三个月了怎么办？", "expected_domain": "劳动法", "golden_docs": ["劳动合同法 第八十五条"]}
```

目标：至少 50 个高质量评估样本，覆盖 5 个法律领域。

### 第二步：定义评估指标

| 指标 | 说明 | 采集方式 |
|---|---|---|
| Retrieval Recall@K | 前 K 个检索结果中包含 golden_docs 的比例 | 自动计算 |
| Domain Accuracy | 领域分类准确率 | 自动计算 |
| Answer Faithfulness | 回答是否忠实于检索材料 | LLM-as-Judge |
| Hallucination Rate | 编造法条/案号的比例 | 人工抽检 + LLM 标记 |
| Citation Accuracy | 引用法条真实存在于知识库的比例 | 自动验证 |
| Avg Response Time | 平均响应耗时 | 日志统计 |

### 第三步：自动化评估脚本

新增：

```text
scripts/evaluate_rag.py
```

运行后输出评估报告：

```
========== RAG 评估报告 ==========
评估数据集：50 个问答对

检索指标：
  Recall@5:    86.0%  (43/50)

分类指标：
  领域准确率：  92.0%

生成指标：
  Faithfulness:      4.2/5.0
  法条幻觉率：     4.0%  (2/50)

建议：
  - 行政法领域 Recall 偏低（75%），建议补充数据
  - 幻觉集中在法条年份/条款号上，建议增加 prompt 约束
================================
```

### 第四步：建立回归基线

每次修改 RAG 链路后，运行评估脚本确保指标不下降。后续可纳入 CI 流程。

## 18.3 修改后的成果

完成后：

- RAG 质量可量化、可追踪。
- 每次修改能判断效果是变好还是变差。
- 法条幻觉率有明确数据，可持续改进。
- 企业验收有数据依据，不再凭感觉。

---

# 第 19 阶段：对话记忆与会话管理（新增）

## 19.1 为什么要改

法律咨询天然是多轮对话。用户会说：

> Q1: "我被公司辞退了。"
> Q2: "公司没给我签合同。"
> Q3: "我工作了一年半，应该赔多少钱？"

如果是无状态 API，第三句 LLM 不知道前文，回答就是孤立的。

当前系统每次请求都是独立的，无法维持对话上下文，导致：
- 用户需要反复描述案情
- LLM 回答缺乏连贯性
- 法律咨询体验差

## 19.2 怎么改

### 第一步：会话历史存储

使用 Redis 存储短期对话历史：

```python
# Key: session:{session_id}
# Value: [{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]
# TTL: 1 小时
```

### 第二步：窗口截断策略

控制上下文窗口，保留：
- 最新的 N 轮对话
- 当前轮次的检索结果
- 系统 prompt

超出窗口时丢弃最早的对话轮次。

### 第三步：API 支持

请求中增加可选字段：

```python
class ChatRequest(BaseModel):
    question: str
    session_id: str | None = None  # 不传则自动创建新会话
```

响应中包含 `session_id`。

### 第四步：会话超时清理

Redis TTL 设为 1 小时，超时后自动清除。
用户无操作 30 分钟后可提示"会话即将过期"。

## 19.3 修改后的成果

完成后：

- 同一 session 内的多轮对话保持上下文连贯。
- 超时后自动清理，不占用存储。
- 法律咨询体验更自然。

---

# 第 20 阶段：CI/CD 自动化管线（新增）

## 20.1 为什么要改

当前项目没有自动化流水线。每次修改代码后：

- 需要手动跑测试
- 需要手动构建镜像
- 需要手动部署到服务器

企业项目的标准是：**提交代码 → 自动测试 → 自动构建 → 自动部署**。

没有 CI/CD，团队协作时容易出现"我本地能跑啊"的问题，也无法保证代码质量。

## 20.2 怎么改

### 第一步：新建 CI 配置文件

新增：

```text
.github/workflows/ci.yml
```

使用 GitHub Actions：

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run unit tests
        run: pytest tests/unit/ --cov=base --cov-report=xml

  docker-build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t fazhitong-api:${{ github.sha }} .
```

### 第二步：CI 中运行的质量门禁

每次提交自动检查：

1. 单元测试是否通过
2. 代码编码是否都是 UTF-8
3. 是否包含明文密钥（通过正则扫描）
4. Docker 镜像是否能构建

### 第三步：后续可扩展 CD

如果需要自动部署到测试环境，可以增加：

```yaml
deploy:
  needs: docker-build
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to test server
      run: |
        ssh user@server "docker compose pull && docker compose up -d"
```

## 20.3 修改后的成果

完成后：

- 代码提交后自动运行测试，问题早发现。
- 主分支代码永远是可部署的状态。
- 新成员加入后提交代码会自动被检查。
- 企业交付更有信心。

---

# 第 21 阶段：个人信息保护合规（新增）

## 21.1 为什么要改

法律咨询系统中，用户可能会输入：

- 姓名、身份证号、手机号
- 家庭住址、公司信息
- 案件细节、合同内容

根据《中华人民共和国个人信息保护法》（PIPL），企业需要：

- 告知用户数据用途
- 限制数据存储期限
- 提供数据删除途径
- 不得将数据用于模型训练
- 确保数据安全

当前系统没有这些措施。

## 21.2 怎么改

### 第一步：数据分类与存储策略

| 数据类型 | 存储位置 | 保留期限 |
|---|---|---|
| 对话记录 | Redis | 1 小时后自动清除 |
| 问答命中统计 | MySQL | 永久（已脱敏） |
| 原始用户输入 | 不入库 | 仅内存处理 |

### 第二步：日志脱敏

已在第 6 阶段实施。确保所有日志中：

- 手机号显示为 `138****5678`
- 身份证显示为 `110101********1234`
- 不记录完整对话原文

### 第三步：用户告知

在前端页面增加隐私提示：

```text
您的输入仅用于本次法律咨询，不会被存储或用于模型训练。
```

### 第四步：数据删除机制

后续可提供接口：

```text
POST /api/v1/user/data/delete
```

用户请求删除后，清除 Redis 中的对话记录。

## 21.3 修改后的成果

完成后：

- 符合 PIPL 基本要求。
- 用户数据不会长期留存。
- 日志不会泄露个人信息。
- 企业合规风险降低。

---

# 上线前检查清单

## 配置检查

- [ ] `.env` 已配置真实环境地址
- [ ] `.env` 没有提交到 Git
- [ ] DashScope API Key 有效
- [ ] MySQL 可连接
- [ ] Redis 可连接
- [ ] Milvus 可连接
- [ ] 模型目录存在
- [ ] 日志目录存在

## 数据检查

- [ ] 问答对已导入 MySQL
- [ ] Redis 缓存可正常读写
- [ ] 法律条文已导入 Milvus
- [ ] 案例库已导入 Milvus
- [ ] 至少测试 10 个法律问题
- [ ] 检索不到时能友好兜底

## API 检查

- [ ] `/api/health` 正常
- [ ] `/api/ready` 正常
- [ ] `/api/chat` 正常
- [ ] 错误请求返回友好错误
- [ ] 请求 ID 正常生成
- [ ] 日志能查到 request_id

## 前端检查

- [ ] 页面无乱码
- [ ] 可以发送问题
- [ ] 回答可以正常展示
- [ ] 引用来源可以展示
- [ ] 移动端布局正常
- [ ] 错误提示友好
- [ ] 免责声明明显

## 安全检查

- [ ] 代码中没有真实 API Key
- [ ] 日志中不会输出 API Key
- [ ] 日志中手机号、身份证已脱敏
- [ ] 管理接口有鉴权
- [ ] CORS 不再使用 `allow_origins=["*"]`，生产环境改为指定域名

## 部署检查

- [ ] Docker 镜像可构建
- [ ] 容器可启动
- [ ] 服务可自动重启
- [ ] 日志可持久化
- [ ] 模型目录挂载正确
- [ ] 备份方案已确认

## 新增：RAG 质量检查

- [ ] 评估数据集已准备（>= 50 条）
- [ ] 检索 Recall@5 >= 80%
- [ ] 法条幻觉率 < 8%
- [ ] 领域分类准确率 >= 85%
- [ ] 评估脚本可一键运行

## 新增：合规检查

- [ ] 每条回答末尾有免责声明
- [ ] 日志已完成脱敏改造
- [ ] 用户数据 TTL 机制已配置
- [ ] 敏感问题触发人工介入提示
- [ ] 前端页面有隐私告知

---

# 建议的实际执行顺序

如果你是小白，建议不要同时改所有东西。

推荐按这个顺序执行：

## 第 1 周：让项目变得安全、可读

1. 新增 `.env.example`
2. 修改 `.gitignore`
3. 改造 `base/config.py` 支持环境变量
4. 修复乱码
5. 新增 README

完成标准：

- 项目没有明文密钥。
- 中文显示正常。
- README 能说明怎么启动。

## 第 2 周：让项目更稳定

1. 改造 `api_server.py` 启动流程
2. 增加 `/api/health` 和 `/api/ready`
3. 统一 API 响应格式
4. 增加错误码
5. 增加日志脱敏

完成标准：

- 服务启动更稳定。
- 出错时不会直接崩。
- 日志可以排查问题。

## 第 3 周：让项目可测试

1. 安装 pytest
2. 新增 `tests/`
3. 写配置测试
4. 写健康检查测试
5. 写文本清洗测试
6. 写 smoke test

完成标准：

- 能运行 `pytest`
- 基础功能有测试保护

## 第 4 周：让项目可部署

1. 新增 Dockerfile
2. 新增 docker-compose.yml
3. 新增部署文档
4. 新增数据初始化文档
5. 跑一遍完整部署流程

完成标准：

- 新机器可以按文档部署。
- 服务可以用 Docker 启动。

## 第 5 周：让项目更像企业产品

1. 前端修复和增强
2. 引用来源展示
3. 管理接口鉴权
4. 监控指标
5. 备份恢复文档

完成标准：

- 可以给企业用户演示。
- 有基本上线能力。

## 第 6 周：RAG 质量保障（新增）

1. 构建评估数据集
2. 实现 RAG 评估脚本
3. 建立指标基线
4. 修复低分领域

完成标准：

- 检索 Recall@5 >= 80%。
- 法条幻觉率 < 8%。

## 第 7 周：对话与合规（新增）

1. 实现会话管理
2. Prompt 合规约束
3. 前端引用展示 + 免责声明
4. CI/CD 管线搭建
5. 个人信息保护措施

完成标准：

- 多轮对话可延续上下文。
- 合规措施完整。
- CI 自动运行。

---

# 最终交付物清单

企业交付时，建议至少包含：

```text
README.md
.env.example
requirements.txt
Dockerfile
docker-compose.yml
docs/deployment.md
docs/data_init.md
docs/api.md
docs/backup_restore.md
tests/
  unit/
  integration/
scripts/check_services.py
scripts/smoke_test.py
scripts/evaluate_rag.py        # 新增：RAG 评估脚本
.github/workflows/ci.yml        # 新增：CI 配置
web/index.html
api_server.py
main.py
base/
rag_qa/
mysql_qa/
case_rag/
```

交付说明中要写清楚：

- 系统功能
- 部署方式
- 初始化数据方式
- API 调用方式
- 常见问题
- 如何备份恢复
- 法律免责声明

---

# 项目达到"企业可用"的判断标准

当下面这些都满足时，可以认为项目基本达到企业内部可用标准：

- [ ] 新人能按照 README 在本地跑起来
- [ ] 服务器能按照部署文档跑起来
- [ ] 配置不写死在代码里
- [ ] 密钥不会提交到仓库
- [ ] 前端中文显示正常
- [ ] 后端接口稳定
- [ ] 数据初始化流程清楚
- [ ] 日志可排查问题
- [ ] 错误返回友好
- [ ] 有基础测试
- [ ] 有 Docker 部署方案
- [ ] 有法律免责声明
- [ ] 有备份恢复方案
- [ ] 至少经过一轮真实问题测试
- [ ] **RAG 评估指标清晰（Recall@K、幻觉率、领域准确率）**
- [ ] **法条幻觉率控制在 8% 以下**
- [ ] **多轮对话可维持上下文**
- [ ] **CI 管线正常运作**
- [ ] **个人信息保护措施到位（脱敏、限时存储、隐私告知）**

---

## 总结

当前项目不是完全不行，它的核心能力已经比较完整。

真正欠缺的是企业项目常见的工程化能力：

- 配置管理
- 安全管理
- 文档
- 测试
- 部署
- 日志
- 错误处理
- 合规提示
- 运维监控
- **RAG 质量保障（新增）**
- **对话管理（新增）**
- **CI/CD 自动化（新增）**
- **个人信息保护（新增）**

建议优先完成：

1. 密钥环境变量化
2. 修复乱码
3. 补 README
4. 优化 API 启动和健康检查
5. 增加最小测试
6. 增加 Docker 部署

这 6 件事做完，这个项目就会从"能跑的 Demo"明显进化为"能交付的工程项目"。
