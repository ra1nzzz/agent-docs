# 管理后台 API 技术选型预研

**任务**: Phase2: 管理后台 API 实现  
**负责人**: yao  
**预研日期**: 2026-03-17  
**状态**: 预研中

---

## 📋 需求回顾

根据任务描述，需要实现以下 4 个核心 API：

1. **Agent 状态 API** - 查询/更新 Agent 运行状态
2. **任务列表 API** - 任务 CRUD、认领、分配
3. **心跳同步 API** - Agent 心跳上报与状态同步
4. **单元测试** - 保证代码质量

---

## 🎯 技术选型

### 1. 运行时环境

| 选项 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Node.js** | 与现有智弈集群技术栈一致，异步 I/O 适合高并发 | 单线程 CPU 密集型任务较弱 | ⭐⭐⭐⭐⭐ |
| Python/FastAPI | 生态丰富，开发效率高 | 与现有技术栈不一致 | ⭐⭐⭐ |
| Go | 性能优秀，并发模型好 | 学习成本，与现有技术栈不一致 | ⭐⭐⭐ |

**结论**: 继续使用 **Node.js 20+**（与智弈集群保持一致）

---

### 2. Web 框架

| 框架 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| **Fastify** | 性能极佳（比 Express 快 2-3 倍），内置 schema 验证，TypeScript 友好 | 生态略小于 Express | ⭐⭐⭐⭐⭐ 推荐 |
| Express | 生态最大，文档丰富，上手简单 | 性能一般，中间件质量参差不齐 | ⭐⭐⭐⭐ 备选 |
| NestJS | 完整的 MVC 架构，依赖注入，TypeScript 原生 | 学习曲线陡峭，样板代码多 | ⭐⭐⭐ 过度设计 |
| Koa | 轻量，async/await 原生支持 | 生态较小，需要自己组装中间件 | ⭐⭐⭐ |

**结论**: 推荐 **Fastify**
- 性能优势明显（适合心跳等高频率请求）
- 内置 JSON Schema 验证（减少依赖）
- 与现有智弈集群代码风格兼容

---

### 3. 数据存储

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **SQLite (better-sqlite3)** | 零配置，单文件，读写性能优秀，支持事务 | 不适合分布式 | ⭐⭐⭐⭐⭐ |
| PostgreSQL | 功能强大，支持复杂查询，分布式友好 | 需要独立部署，运维成本 | ⭐⭐⭐⭐ |
| MongoDB | 文档型，灵活，与 Node.js 集成好 | 事务支持较弱，资源占用高 | ⭐⭐⭐ |
| 文件系统 (JSON) | 零依赖，调试方便 | 并发写入有问题，无事务 | ⭐⭐ |

**结论**: 推荐 **SQLite**
- 智弈集群当前使用文件系统存储，SQLite 是自然升级
- 零配置，单文件部署
- 支持事务，适合任务状态变更
- 查询性能远优于 JSON 文件遍历

---

### 4. 认证授权

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Token 验证 (现有)** | 与智弈集群现有机制一致，简单直接 | 需要自行实现 Token 管理 | ⭐⭐⭐⭐⭐ |
| JWT | 无状态，可扩展 | 需要密钥管理，token 吊销麻烦 | ⭐⭐⭐⭐ |
| Session | 成熟，支持吊销 | 需要存储，分布式需要共享 session | ⭐⭐⭐ |

**结论**: 沿用智弈集群现有的 **X-Claw-Token + X-Claw-Agent** 头部验证机制

---

### 5. 单元测试框架

| 框架 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **Vitest** | 速度快，Jest 兼容 API，原生 ESM 支持 | 相对较新 | ⭐⭐⭐⭐⭐ |
| Jest | 生态最大，功能完整 | 配置复杂，ESM 支持一般 | ⭐⭐⭐⭐ |
| Mocha + Chai | 灵活，可组合 | 需要自行组装，配置繁琐 | ⭐⭐⭐ |
| Node.js test runner | 原生无需依赖 | 功能相对简单 | ⭐⭐⭐ |

**结论**: 推荐 **Vitest**
- 配置简单，开箱即用
- 与现有智弈集群测试工具兼容
- 支持覆盖率报告

---

### 6. API 文档

| 方案 | 优势 | 劣势 | 推荐度 |
|------|------|------|--------|
| **OpenAPI + Swagger UI** | 行业标准，自动生成文档，支持在线测试 | 需要额外依赖 | ⭐⭐⭐⭐⭐ |
| 手写 Markdown | 零依赖，灵活 | 无法自动同步代码变更 | ⭐⭐⭐ |
| Postman Collection | 易于分享和测试 | 文档与代码分离 | ⭐⭐⭐ |

**结论**: 推荐 **OpenAPI (Swagger)**
- Fastify 有成熟的 `@fastify/swagger` 插件
- 自动生成文档，减少维护成本
- 便于团队协作和接口对齐

---

## 📦 推荐技术栈汇总

```
┌─────────────────────────────────────────────────────────┐
│                    管理后台 API 技术栈                    │
├─────────────────────────────────────────────────────────┤
│ 运行时：Node.js 20+                                     │
│ 框架：Fastify 4.x                                       │
│ 数据库：SQLite (better-sqlite3)                         │
│ 认证：X-Claw-Token / X-Claw-Agent (现有机制)            │
│ 测试：Vitest                                            │
│ 文档：OpenAPI + Swagger UI                              │
│ 部署：PM2 (与智弈集群一致)                              │
└─────────────────────────────────────────────────────────┘
```

---

## 🗂️ 项目结构建议

```
agent-admin-api/
├── src/
│   ├── index.js           # 入口文件
│   ├── config.js          # 配置管理
│   ├── db.js              # SQLite 数据库连接
│   ├── schema.js          # 数据库表结构
│   ├── routes/
│   │   ├── agents.js      # Agent 状态 API
│   │   ├── tasks.js       # 任务管理 API
│   │   └── heartbeat.js   # 心跳同步 API
│   ├── middleware/
│   │   └── auth.js        # Token 验证中间件
│   └── utils/
│       └── webhook.js     # Webhook 通知工具
├── test/
│   ├── agents.test.js
│   ├── tasks.test.js
│   └── heartbeat.test.js
├── docs/
│   └── openapi.yaml       # OpenAPI 规范
├── package.json
└── README.md
```

---

## 📊 核心 API 设计草案

### 1. Agent 状态 API

```
GET    /api/agents              # 获取所有 Agent 状态
GET    /api/agents/:id          # 获取指定 Agent 状态
PUT    /api/agents/:id/status   # 更新 Agent 状态
POST   /api/agents/register     # 注册新 Agent
```

### 2. 任务管理 API

```
GET    /api/tasks               # 获取任务列表（支持筛选）
GET    /api/tasks/:id           # 获取任务详情
POST   /api/tasks               # 创建任务
PUT    /api/tasks/:id           # 更新任务
DELETE /api/tasks/:id           # 删除任务
POST   /api/tasks/:id/claim     # 认领任务
POST   /api/tasks/:id/pause     # 暂停任务
POST   /api/tasks/:id/resume    # 恢复任务
```

### 3. 心跳同步 API

```
POST   /api/heartbeat           # Agent 心跳上报
GET    /api/heartbeat/status    # 获取心跳状态汇总
```

---

## ⏭️ 下一步行动

1. **等待 Phase1 输出**
   - 开源模型调研报告（xiaoshen）
   - API 设计文档（lucy）

2. **预研完成后**
   - 搭建项目骨架
   - 实现基础 CRUD
   - 编写单元测试
   - 生成 API 文档

3. **预计时间线**
   - Phase1 完成：03-19
   - API 实现开始：03-20
   - 初步完成：03-22

---

## 📝 备注

- 本预研基于当前对任务的理解，具体实现需根据 Phase1 的 API 设计文档调整
- 技术选型优先考虑与现有智弈集群技术栈的一致性，降低维护成本
- 所有代码需包含单元测试，目标覆盖率 ≥80%

---

**文档版本**: v1.0  
**最后更新**: 2026-03-17  
**作者**: yao (星瑶 ✨)
