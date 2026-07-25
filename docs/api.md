# 律所法律知识库 RAG API 接口文档

## V2 主接口

- `POST /api/v1/search`：统一检索法规、司法解释、案例和内部资料。
- `POST /api/v1/ask`：生成带 `citation_id`、`document_id`、`chunk_id` 和原文的研究辅助回答。
- `GET|POST /api/v1/documents`：查询或上传知识库文件。
- `GET|DELETE /api/v1/documents/{document_id}`：查看或删除文档。
- `POST /api/v1/documents/{document_id}/reindex`：解析、切分、向量化并重新索引文档。

完整请求和响应结构以服务启动后的 `/docs` OpenAPI 页面为准。下方 `/api/v1/chat` 内容是兼容旧客户端的接口说明，新客户端应使用 `/api/v1/ask`。

## 基础信息

- **Base URL**: `http://localhost:8000`
- **当前版本**: `/api/v1/`（旧路由 `/api/` 保持兼容）
- **格式**: JSON
- **文档**: 启动后访问 `http://localhost:8000/docs`（Swagger UI）

---

## POST /api/v1/chat

法律问答聊天接口。

### 请求

```json
{
    "question": "劳动法规定的工作时间是多少？",
    "source_filter": null
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `question` | string | 是 | 法律问题，1-2000 字 |
| `source_filter` | string 或 null | 否 | 限定检索领域，如 `"劳动法"`；`null` 表示自动分类 |

### 响应

```json
{
    "answer": "根据《中华人民共和国劳动法》第三十六条...",
    "source_filter": "劳动法",
    "request_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "used_rag": true,
    "used_fallback": false,
    "references": [
        {
            "type": "article",
            "title": "中华人民共和国劳动法",
            "source": "法律条文",
            "content": "第三十六条 国家实行劳动者每日工作时间不超过八小时...",
            "score": 0.92
        }
    ],
    "disclaimer": "以上内容由 AI 生成，仅供参考，不构成正式法律意见。"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `answer` | string | 法律回答（Markdown） |
| `source_filter` | string 或 null | 当前生效的法律领域 |
| `request_id` | string | 请求唯一 ID，排查日志用 |
| `used_rag` | bool | 是否使用了 RAG 检索 |
| `used_fallback` | bool | 是否发生了降级兜底 |
| `references` | array | 引用的法律条文 / 案例列表 |
| `disclaimer` | string | 免责声明 |

---

## GET /api/v1/health

健康检查（存活）。

### 响应

```json
{
    "status": "ok",
    "ready": true,
    "version": "1.0.0",
    "timestamp": "2026-07-06T10:00:00Z"
}
```

---

## GET /api/v1/ready

就绪检查（依赖是否全部就绪），用于负载均衡判断。

### 响应（就绪）

```json
{
    "status": "ready",
    "ready": true
}
```

### 响应（未就绪）

HTTP 503

```json
{
    "status": "starting",
    "ready": false
}
```

---

## 错误响应格式

```json
{
    "code": "ERROR_CODE",
    "message": "人类可读的错误描述",
    "request_id": "a1b2c3d4..."
}
```

### 错误码

| 错误码 | HTTP 状态码 | 含义 |
|---|---|---|
| `INVALID_REQUEST` | 400 | 请求参数错误（如 question 为空或超长） |
| `SERVICE_NOT_READY` | 503 | 服务尚未初始化完成 |
| `RATE_LIMITED` | 429 | 请求过于频繁 |
| `LLM_ERROR` | 502 | 大模型调用失败 |
| `RETRIEVAL_ERROR` | 502 | 知识检索失败 |
| `INTERNAL_ERROR` | 500 | 系统内部错误 |

---

## 兼容旧路由

以下旧路由依然可用：

| 旧路由 | 新路由 |
|---|---|
| `POST /api/chat` | `POST /api/v1/chat` |
| `GET /api/health` | `GET /api/v1/health` |

---

## GET /metrics

监控指标端点。

### 响应

```json
{
    "request_count_total": 1024,
    "error_count_total": 12,
    "average_latency_ms": 1250.5
}
```

---

## POST /api/v1/user/data/delete

根据 PIPL（个人信息保护法）要求，删除用户对话数据。

需要 Token 认证（如已配置 `API_TOKEN`）。

### 请求

```json
{
    "session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

### 响应

```json
{
    "status": "deleted",
    "session_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

## 认证方式

如果 `.env` 中配置了 `API_TOKEN`，管理接口需要 Bearer Token：

```
Authorization: Bearer your_token_here
```

受保护的接口：
- `POST /api/v1/admin/qa`
- `PUT /api/v1/admin/qa/{id}`
- `DELETE /api/v1/admin/qa/{id}`
- `POST /api/v1/user/data/delete`

未配置 `API_TOKEN` 时，认证跳过（兼容现有部署）。

---

## 会话管理

`POST /api/v1/chat` 支持 `session_id` 字段：

```json
{
    "question": "我被公司辞退了",
    "session_id": "a1b2c3d4-..."         // 不传则自动创建新会话
}
```

响应中包含 `session_id`，可用于多轮对话追踪。
会话数据在 Redis 中保留 1 小时后自动清除。

---

## 前端页面

访问 `http://localhost:8000` 即可打开 Web 聊天界面。
