# 管理后台 API 技术选型预研

**任务**: Phase2: 管理后台 API 实现  
**负责人**: yao  
**预研日期**: 2026-03-17  
**状态**: 预研中

---

## 📋 需求回顾

根据任务描述，需要实现以下 4 个核心 API:

1. **Agent 状态 API** - 查询/更新 Agent 运行状态
2. **任务列表 API** - 任务 CRUD、认领、分页
3. **心跳同步 API** - Agent 心跳上报与状态同步
4. **单元测试** - 保证代码质量

---

## 🎯 技术选型

### 1. 运行时环境

| 选项 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Node.js** | 与现有智弈集群技术栈一致，异步 I/O 适合高并发 | 单线程 CPU 密集型任务受限 | 🦀🦀🦀🦀🦀 |
| Python/FastAPI | 开发效率高，异步支持好 | 与现有技术栈不一致 | 🦀🦀🦀 |
| Go | 高性能，并发模型优秀 | 学习曲线较陡，与现有不一致 | 🦀🦀 |

**结论**: 继续使用 **Node.js 20+**（与智弈集群保持一致）

---

### 2. Web 框架

| 框架 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Fastify** | 高性能优化（比 Express 快 2-3 倍），内置 JSON Schema 验证，TypeScript 友好 | 生态相对 Express 较小 | 🦀🦀🦀🦀🦀 |
| Express | 生态最丰富，文档海量，上手快 | 性能一般，中间件质量参差不齐 | 🦀🦀🦀🦀 |
| NestJS | 规范的 MVC 架构，DI 容器，TypeScript 原生 | 学习曲线较陡，较重 | 🦀🦀🦀 |
| Koa | 轻量，async/await 原生支持 | 生态较小，需要自己组装中间件 | 🦀🦀 |

**结论**: 推荐 **Fastify**
- 性能优化明显，适合高频 API 交互
- 内置 JSON Schema 验证（减少依赖）
- 与现有智弈集群框架一致

---

### 3. 数据存储

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **SQLite (better-sqlite3)** | 零配置，单文件，适合原型开发，性能优秀 | 不适合大规模分布式 | 🦀🦀🦀🦀🦀 |
| PostgreSQL | 功能强大，支持复制，适合生产 | 需要单独部署，重量级 | 🦀🦀🦀🦀 |
| MongoDB | 文档型，灵活，与 Node.js 天然契合 | 服务端资源消耗高，付费版本贵 | 🦀🦀🦀 |
| 文件系统 (JSON) | 零配置，便于调试 | 并发写入问题，不适合生产 | 🦀 |

**结论**: 推荐 **SQLite**
- 与当前智弈集群一致（目前用文件系统，需升级）
- 零配置，单文件便于备份和迁移
- 性能足够（单实例几千 QPS）
- 查询灵活，支持 JSON 文件直接查询

---

### 4. 认证授权

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Token 认证 (现有)** | 与智弈集群现有方案一致，无缝集成 | 需要自行管理 Token 生命周期 | 🦀🦀🦀🦀🦀 |
| JWT | 无状态，可自包含用户信息 | 需要处理 token 刷新，黑名单难 | 🦀🦀🦀 |
| Session | 简单，服务器端可控 | 需要存储 session，分布式需共享存储 | 🦀🦀 |

**结论**: 沿用现有 **X-Claw-Token + X-Claw-Agent** 双向认证

---

### 5. 单元测试

| 框架 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Vitest** | 速度极快，Jest 兼容 API，原生 ESM 支持 | 相对较新 | 🦀🦀🦀🦀🦀 |
| Jest | 生态最丰富，功能全面 | 配置稍重，ESM 支持一般 | 🦀🦀🦀🦀 |
| Mocha + Chai | 灵活，可定制 | 需要自己配置断言、覆盖率等 | 🦀🦀 |
| Node.js test runner | 原生，无需依赖 | 功能相对简单 | 🦀🦀 |

**结论**: 推荐 **Vitest**
- 配置简单，开箱即用
- 与 Fastify 生态兼容好
- 支持快照测试和并发

---

### 6. API 文档

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **OpenAPI + Swagger UI** | 行业标准，可自动生成 UI，支持测试 | 需要编写 YAML/JSON 注释 | 🦀🦀🦀🦀🦀 |
| 手写 Markdown | 简单，版本控制友好 | 无法自动同步代码变更 | 🦀🦀 |
| Postman Collection | 便于团队协作和测试 | 与代码分离，维护成本高 | 🦀🦀 |

**结论**: 推荐 **OpenAPI (Swagger)**
- Fastify 有成熟的 `@fastify/swagger` 插件
- 自动从代码生成文档，减少人工维护
- 便于前后端协作和联调

---

## 🎨 推荐技术栈总结

```
🦀 运行时:       Node.js 20+
🦀 框架:         Fastify 4.x
🦀 数据库:       SQLite (better-sqlite3)
🦀 认证:         X-Claw-Token / X-Claw-Agent (现有)
🦀 测试:         Vitest
🦀 文档:         OpenAPI + Swagger UI
🦀 部署:         Docker (Node.js 20+ alpine)
```

---

## 📁 项目结构建议

```
agent-admin-api/
🦀 src/
    🦀 index.js           # 应用入口
    🦀 config.js          # 配置管理
    🦀 db.js              # SQLite 连接管理
    🦀 routes/
    🦀   agents.js        # Agent 状态 API
    🦀   tasks.js         # 任务管理 API
    🦀   heartbeat.js     # 心跳同步 API
    🦀 middleware/
    🦀   auth.js          # Token 验证中间件
    🦀   validation.js    # 请求体验证
    🦀 utils/
    🦀   logger.js        # 日志工具
🦀 test/
    🦀 agents.test.js
    🦀 tasks.test.js
    🦀 heartbeat.test.js
🦀 docs/
    🦀 openapi.yaml       # OpenAPI 规范文件
🦀 docker-compose.yml
🦀 Dockerfile
🦀 package.json
🦀 README.md
```

---

## 📡 核心 API 设计

### 1. Agent 状态 API

```
GET    /api/agents              # 获取所有 Agent 状态
GET    /api/agents/:id          # 获取指定 Agent 状态
PUT    /api/agents/:id/status   # 更新 Agent 状态
POST   /api/agents/register     # 注册新 Agent
```

**示例响应**:
```json
{
  "agents": [
    {
      "id": "xiaoshen",
      "name": "小深",
      "role": "analyst",
      "status": "online",
      "last_heartbeat": "2026-03-17T01:23:00Z",
      "current_task": "Phase1 开源模块调研"
    }
  ]
}
```

---

### 2. 任务管理 API

```
GET    /api/tasks               # 获取任务列表（支持分页、过滤）
GET    /api/tasks/:id           # 获取任务详情
POST   /api/tasks               # 创建任务
PUT    /api/tasks/:id           # 更新任务
DELETE /api/tasks/:id           # 删除任务
POST   /api/tasks/:id/claim     # 认领任务
POST   /api/tasks/:id/pause     # 暂停任务
POST   /api/tasks/:id/resume    # 恢复任务
```

**示例 - 认领任务**:
```json
POST /api/tasks/task_xxx/claim
{
  "agent_id": "xiaoshen",
  "estimated_days": 1
}
```

---

### 3. 心跳同步 API

```
POST   /api/heartbeat           # Agent 心跳上报
GET    /api/heartbeat/status    # 获取心跳状态统计
```

**心跳上报示例**:
```json
POST /api/heartbeat
{
  "agent_id": "xiaoshen",
  "status": "online",
  "commit_url": "https://github.com/.../commit/abc123"
}
```

---

## 🔗 下一步计划

1. **等待 Phase1 输出** - 审阅 Lucy 的 API 设计文档（本阶段）
2. **编码实现** - 完成核心 CRUD 和认证
3. **单元测试** - 覆盖率达 80%+
4. **API 文档生成** - 自动生成 OpenAPI 规范
5. **部署验证** - Docker 部署并测试

---

## ⏰ 时间线

- **Phase1 完成**: 03-19
- API 实现启动: 03-20
- 第一阶段完成: 03-22

---

**文档版本**: v1.0  
**最后更新**: 2026-03-17  
**作者**: yao (Lucy)

---

## 🔍 审阅意见（小深 - 2026-03-17）

### 总体评价
✅ **设计优秀，具备了工程化的基础架构**  
技术选型简洁合理，API 结构清晰，测试策略明确。可以立即开始开发。

### 优点
1. **技术栈轻量**：Node.js + Fastify + SQLite，与团队现有技能栈一致
2. **API 结构清晰**：三大核心资源（Agents, Tasks, Heartbeat）划分合理
3. **测试策略明确**：Vitest + 80%+ 覆盖率目标
4. **文档自动化**：OpenAPI/Swagger 自动生成
5. **项目结构规范**：分层架构清晰

---

### ⚠️ 待优化项

#### 1. 🔐 认证授权加固
**现状**：仅依赖 `X-Claw-Token` 单因素认证  
**风险**：Token 泄露即系统沦陷，无细粒度权限

**建议**：
- 引入双 Token 机制：`X-Claw-Token`（Agent认证）+ `X-Claw-Session`（用户会话）
- 增加 Scope 权限控制（如 `admin:agents`、`read:heartbeat`）
- 实现 Token 轮转与撤销端点

#### 2. 📊 数据存储演进路径
**现状**：SQLite 单一存储  
**问题**：并发写入瓶颈、无内置备份、不适合多实例扩展

**优化建议**：
- **Phase1（当前）**：SQLite ✅ 可接受（单实例，小团队）
- **Phase2（3-6个月）**：迁移到 **PostgreSQL**（与 Phase1 调研推荐一致）
  - 支持并发读写，无锁竞争
  - 内置复制与备份
  - 与 Redash 共享数据库，减少基础设施
- **预留数据层抽象**：使用 Repository 模式，未来切换数据库只需替换实现

#### 3. 📈 可观测性集成（Monitoring）
**现状**：无指标暴露

**建议**：基于 **Prometheus + Grafana** 监控

```javascript
// 使用 prom-client 暴露 /metrics
fastify.register(require('fastify-metrics'), {
  endpoint: '/metrics',
  promClient: client
});
```

**关键指标**：
- 请求量 (QPS) 和延迟 (p50/p95/p99)
- 错误率 (4xx/5xx)
- Agent 心跳间隔分布
- SQLite 连接池使用率

#### 4. 🔄 任务调度升级
**现状**：内置简单定时任务  
**问题**：单进程调度，无持久化、无分布式协调

**优化方向**：
- **短期**（Phase1 内）：继续使用内置调度，满足基本需求
- **长期**（Phase2）：集成 **Agenda**（Phase1 推荐方案）
  - 优势：MongoDB 持久化，支持分布式锁，cron 语法丰富
  - 迁移路径：将定时任务定义迁移到 Agenda，API 保持兼容

#### 5. 🛡️ API 安全深化
| 措施 | 建议 | 理由 |
|------|------|------|
| 速率限制 | 每 Agent/IP 100 req/min | 防刷 |
| CORS | 配置白名单（仅管理后台域名） | 防跨域 |
| 请求体大小限制 | 1MB | 基础防护 |
| 日志脱敏 | 响应中不泄露 Token | 防日志泄露 |
| HTTPS 强制 | 生产环境禁用 HTTP | 传输安全 |

#### 6. 📚 API 设计细化
- 使用 `/api/v1/` 前缀，便于未来版本迁移
- 资源 ID 使用 UUID 而非自增整数（防枚举）
- 错误响应标准化：

```json
{
  "error": {
    "code": "AGENT_NOT_FOUND",
    "message": "Agent not found",
    "details": { "agent_id": "xxx" }
  }
}
```

- 分页支持：`GET /api/tasks?status=pending&page=1&limit=20`

#### 7. 🐳 容器化部署
建议提供 Dockerfile 和 docker-compose.yml，并使用环境变量配置敏感信息。

---

## 🔮 演进路线图

| 阶段 | 时间 | 重点 | 关联 Phase1 推荐 |
|------|------|------|-----------------|
| Phase A（当前） | 3-20 ~ 3-22 | 核心 CRUD + 认证 + 测试 | - |
| Phase B（1个月后） | 4月 | 迁移到 PostgreSQL + 备份策略 | 与 Redash 共享 PG |
| Phase C（2个月后） | 5月 | 集成 Agenda 任务调度 | Agenda 轻量定时任务 |
| Phase D（3个月后） | 6月 | Prometheus metrics + Grafana Dashboard | 监控告警体系 |
| Phase E（6个月后） | - | 根据规模评估是否需要日志聚合 | Loki/ELK 按需 |

---

## 📌 具体修改建议

### 项目结构调整建议
```
agent-admin-api/
├── src/
│   ├── index.js           # 启动入口
│   ├── config/            # 配置管理 (env 转换)
│   ├── db/                # 数据层 (Repository 模式)
│   ├── routes/
│   │   ├── agents.js
│   │   ├── tasks.js
│   │   └── heartbeat.js
│   ├── middleware/
│   │   ├── auth.js        # Token 验证 + Scope
│   │   ├── rate-limit.js  # 速率限制
│   │   └── validation.js  # 请求体校验
│   ├── utils/
│   │   ├── metrics.js     # Prometheus 指标
│   │   └── logger.js      # 结构化日志
│   └── plugins/           # Fastify 插件封装
├── test/                  # 单元 + 集成测试
├── docs/                  # OpenAPI + 部署文档
├── docker-compose.yml
├── Dockerfile
└── package.json
```

### 代码示例（metrics.js）
```javascript
const client = require('prom-client');

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 1.5, 2, 5]
});

module.exports = function registerMetrics(fastify) {
  fastify.addHook('onSend', (req, reply, payload) => {
    httpRequestDuration.observe({
      method: req.method,
      route: req.route.path,
      status: reply.statusCode
    });
  });

  fastify.get('/metrics', async () => client.register.metrics());
};
```

---

**审阅人**: 小深 (xiaoshen)  
**日期**: 2026-03-17  
**关联任务**: Phase1 开源模块调研完成，Phase2 工作后台 API 实现启动
