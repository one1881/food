# 合同智能生成与法务审核工作台 - API 文档

**文档版本：** v3.0（A2A + Skills + MCP 融合版）
**创建日期：** 2026-08-19
**API 风格：** RESTful
**基础路径：** `/api/v1`
**最后更新：** 2026-08-19

---

## 1. API 设计原则

- ✅ **RESTful 风格** — 资源化、语义化
- ✅ **统一响应格式** — 成功 / 失败格式一致
- ✅ **A2A 透明** — 前端无需感知 Agent 架构
- ✅ **异步任务模式** — AI 生成采用异步 + 轮询状态

---

## 2. 通用规范

### 2.1 请求头
```
Authorization: Bearer {token}
Content-Type: application/json
```

### 2.2 响应格式

**成功响应：**
```json
{ "code": 0, "message": "success", "data": { ... } }
```

**失败响应：**
```json
{ "code": 40001, "message": "参数错误", "detail": "合同标题不能为空" }
```

---

## 3. 认证接口

### 3.1 用户登录

**`POST /api/v1/auth/login`**

请求：
```json
{ "username": "zhangsan", "password": "password123" }
```

响应：
```json
{
  "code": 0,
  "data": {
    "token": "eyJhbG...VCJ9...",
    "user": { "id": 2, "username": "zhangsan", "real_name": "张三", "role": "drafter" }
  }
}
```

---

## 4. 合同接口（V1 核心 + V2 扩展）

### 4.1 创建合同（AI 生成，异步）

**`POST /api/v1/contracts`**

请求：
```json
{
  "title": "与A公司销售合同",
  "contract_type": "销售合同",
  "customer_name": "A公司",
  "amount": 1000000,
  "term": "1年",
  "special_requirements": "需要添加违约责任条款"
}
```

响应（立即返回任务 ID）：
```json
{
  "code": 0,
  "message": "合同生成中，请稍候...",
  "data": {
    "contract_id": 123,
    "task_id": "task_abc123",
    "estimated_time": 180,
    "status_url": "/api/v1/agent/tasks/task_abc123/status"
  }
}
```

**后台 A2A + Skills + MCP 流程：**
```
主控 Agent 接收任务
  → 数据接入子 Agent（并行 3 Skills，经数据源 MCP）约 5 秒
  → 合同生成子 Agent（经合同知识 MCP 取模板，调用大模型）约 2-4 分钟
  → 合规审核子 Agent（并行 3 Skills，经合规审核 MCP）约 10 秒
  → 主控 Agent 聚合结果 → 保存数据库
```

### 4.2 查询 Agent 任务状态（轮询）

**`GET /api/v1/agent/tasks/{task_id}/status`**

响应（生成中）：
```json
{
  "code": 0,
  "data": {
    "task_id": "task_abc123",
    "status": "running",
    "progress": 60,
    "current_stage": "合同生成中...",
    "agents": {
      "data_agent": { "status": "completed", "duration": 5.2 },
      "generation_agent": { "status": "running", "progress": 60 },
      "compliance_agent": { "status": "pending" }
    }
  }
}
```

响应（生成完成）：
```json
{
  "code": 0,
  "data": {
    "task_id": "task_abc123",
    "status": "completed",
    "progress": 100,
    "result": {
      "contract_id": 123,
      "version_id": 1,
      "content": "<h1>销售合同</h1><p>第一条...</p>",
      "generation_context": { "customer_data": {}, "historical_contracts": [], "regulations": [] },
      "compliance_result": { "risks": [], "score": 85 }
    },
    "execution_time": 245.5
  }
}
```

### 4.3 查询 Agent 执行日志（可追溯）

**`GET /api/v1/agent/tasks/{task_id}/logs`**

响应：
```json
{
  "code": 0,
  "data": {
    "task_id": "task_abc123",
    "logs": [
      { "agent": "data_agent", "skill": "get_customer_info", "status": "success", "duration_ms": 1250, "input": {}, "output": {} },
      { "agent": "generation_agent", "skill": "generate", "status": "success", "duration_ms": 245800, "input": {}, "output": {} }
    ]
  }
}
```

### 4.4 获取合同列表

**`GET /api/v1/contracts`**

查询参数：`?page=1&page_size=20&contract_type=销售合同&status=draft&keyword=A公司`

### 4.5 重新生成合同（V2 增强）

**`POST /api/v1/contracts/{id}/regenerate`**

请求：
```json
{
  "feedback": "需要更专业的法律用语",
  "adjust_params": { "tone": "formal", "detail_level": "high" }
}
```

---

## 5. 智能推荐接口（V2）

**`GET /api/v1/contracts/{id}/recommendations`**

响应：
```json
{
  "code": 0,
  "data": {
    "similar_cases": [ { "id": 101, "title": "与B公司销售合同", "similarity": 0.85, "generated_by": "recommend_agent.suggest_similar_cases" } ],
    "standard_clauses": [ { "title": "标准违约责任条款", "content": "...", "relevance_score": 92, "generated_by": "recommend_agent.suggest_standard_clauses" } ],
    "risk_warnings": [ { "type": "付款条款不明确", "severity": "medium", "suggestion": "...", "generated_by": "recommend_agent.predict_risks" } ]
  }
}
```

---

## 6. 批注接口（V2）

**`POST /api/v1/contracts/{id}/annotations`**

请求：
```json
{
  "version_id": 1,
  "anchor_text": "第三条 合同金额",
  "anchor_position": 1250,
  "content": "建议明确付款时间",
  "annotation_type": "suggestion"
}
```

---

## 7. 完整接口清单

### V1 核心接口（20+）

| 模块 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 认证 | POST | `/api/v1/auth/login` | 登录 |
| | GET | `/api/v1/auth/me` | 当前用户 |
| 合同 | POST | `/api/v1/contracts` | 创建合同（异步） |
| | GET | `/api/v1/contracts` | 合同列表 |
| | GET | `/api/v1/contracts/{id}` | 合同详情 |
| | PUT | `/api/v1/contracts/{id}` | 更新合同 |
| | DELETE | `/api/v1/contracts/{id}` | 删除合同 |
| | POST | `/api/v1/contracts/{id}/submit` | 提交审核 |
| | GET | `/api/v1/contracts/{id}/export/word` | 导出 Word |
| Agent | GET | `/api/v1/agent/tasks/{task_id}/status` | 任务状态 |
| | GET | `/api/v1/agent/tasks/{task_id}/logs` | 任务日志 |
| | GET | `/api/v1/agent/stats` | Agent 统计 |
| | POST | `/api/v1/agent/tasks/{task_id}/retry` | 重试任务 |
| 审核 | GET | `/api/v1/reviews/pending` | 待审核列表 |
| | POST | `/api/v1/reviews` | 提交审核意见 |
| 模板 | GET | `/api/v1/templates` | 模板列表 |
| | POST | `/api/v1/templates` | 创建模板 |

### V2 扩展接口（12+）

| 模块 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 智能推荐 | GET | `/api/v1/contracts/{id}/recommendations` | 获取推荐 |
| | POST | `/api/v1/contracts/{id}/regenerate` | 重新生成 |
| 批注 | POST | `/api/v1/contracts/{id}/annotations` | 添加批注 |
| | GET | `/api/v1/contracts/{id}/annotations` | 批注列表 |
| | PUT | `/api/v1/annotations/{id}` | 更新批注 |
| | DELETE | `/api/v1/annotations/{id}` | 删除批注 |
| 文档 | GET | `/api/v1/contracts/{id}/export/pdf` | 导出 PDF |
| | GET | `/api/v1/contracts/{id}/versions/compare` | 版本对比 |
| 统计 | GET | `/api/v1/stats/dashboard` | 数据统计 |

---

## 8. A2A 架构对前端透明

```
前端调用：POST /api/v1/contracts
  → FastAPI 立即返回 task_id
  → 前端轮询 GET /api/v1/agent/tasks/{task_id}/status
  → 前端收到：完整合同 + 生成依据
```

**后端 A2A + Skills + MCP 流程（前端无感知）：**
```
主控 Agent → 数据接入子 Agent（3 Skills，经数据源 MCP）
          → 合同生成子 Agent（经合同知识 MCP）
          → 合规审核子 Agent（3 Skills，经合规审核 MCP）
          → [V2] 智能推荐子 Agent
          → 结果聚合 → 保存数据库
```

---

## 9. 错误码定义

| 错误码 | 说明 |
|--------|------|
| 0 | 成功 |
| 40001 | 参数错误 |
| 40101 | 未认证 |
| 40301 | 无权限 |
| 40401 | 资源不存在 |
| 50001 | 服务器错误 |
| 50002 | Agent 执行失败 |
| 50003 | 大模型 API 超时 |

---

**文档状态：** ✅ 已完成
