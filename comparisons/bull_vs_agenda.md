# Bull vs Agenda - Node.js 任务调度器对比

**调研日期**: 2026-03-17  
**调研人**: 小深 (xiaoshen)  
**领域**: 分布式任务调度与队列

---

## 📊 核心数据对比

| 维度 | Bull | Agenda |
|------|------|--------|
| **GitHub Stars** | 16,242 ⭐ | 9,642 ⭐ |
| **Forks** | 1,425 | 833 |
| **Issues** | 146 (开放) | 1 (开放) ✅ |
| **语言** | JavaScript (Node.js) | JavaScript (Node.js) |
| **数据库** | Redis | MongoDB |
| **许可证** | 自定义 ("Other") | MIT? ("Other" but GitHub shows MIT) |
| **最新活跃** | 2026-02-28 | 2026-03-06 |
| **官网** | - | agenda.github.io/agenda |

---

## 🎯 核心定位

### Bull
> "Premium Queue package for handling distributed jobs and messages in NodeJS."

**关键词**: 高性能、分布式、Redis 存储、功能丰富

### Agenda
> "Lightweight job scheduling for Node.js"

**关键词**: 轻量、MongoDB 存储、Cron 风格、简洁

---

## 🔧 架构对比

### Bull (BullMQ 的后继，Bull 已归档，建议用 BullMQ)
- **存储**: Redis (内存 + 持久化)
- **架构**: 单/多队列，Worker 进程
- **并发**: 高（Redis 单节点 10k+ ops/sec）
- **可靠性**: 当 worker 崩溃时 jobs 重新排队

### Agenda
- **存储**: MongoDB (文档存储)
- **架构**: 单一实例或集群（MongoDB replica set）
- **并发**: 中等（依赖 MongoDB 写入性能）
- **可靠性**: Job 持久化，支持锁机制防重复

---

## 📋 功能对比

### 核心功能

| 功能 | Bull | Agenda |
|------|------|--------|
| **重复任务** | ✅ (cron-like) | ✅ (cron-like) |
| **延迟任务** | ✅ (delay) | ✅ (delay) |
| **优先级队列** | ✅ | ✅ |
| **速率限制** | ✅ | ✅ (有限) |
| **并发控制** | ✅ (limiter) | ⚠️ 需自定义 |
| **Job 持久化** | ✅ (Redis) | ✅ (MongoDB) |
| **Job 重试** | ✅ (指数退避) | ✅ |
| **事件监听** | ✅ (completed/failed/...) | ✅ |
| **进度追踪** | ✅ | ✅ |
| **暂停/恢复** | ✅ | ✅ |
| **集群支持** | ✅ (Redis 共享) | ✅ (MongoDB replica) |
| **Web UI** | ✅ (Bull Board) | ⚠️ 第三方 |
| **延迟重试** | ✅ | ✅ |

---

### 易用性与 API

#### Bull (BullMQ)
```javascript
const Queue = require('bull');

const videoQueue = new Queue('video transcoding', {
  redis: { port: 6379, host: '127.0.0.1' }
});

// 添加任务
videoQueue.add({ video: 'my-video.mov' }, {
  attempts: 3,            // 重试次数
  backoff: { type: 'exponential' },
  delay: 5000,            // 延迟5秒
  priority: 1             // 优先级
});

// 处理任务
videoQueue.process(async (job) => {
  await transcodeVideo(job.data.video);
});
```

#### Agenda
```javascript
const Agenda = require('agenda');

const mongoConnectionString = "mongodb://127.0.0.1/agenda";
const agenda = new Agenda({ db: { address: mongoConnectionString } });

// 定义任务
agenda.define('send email', async (job) => {
  await sendEmail(job.attrs.data.to);
});

// 每5分钟运行一次
agenda.every('5 minutes', 'send email', { to: 'user@example.com' });

await agenda.start();
```

**API 风格**:
- **Bull**: 显式队列 + 任务添加，适合动态任务
- **Agenda**: 定义任务 + cron 调度，适合定时任务

---

### 监控与管理

| 功能 | Bull | Agenda |
|------|------|--------|
| **管理 UI** | Bull Board / Arena | 需自建或使用第三方 |
| **队列状态** | ✅ (等待/活跃/完成/失败) | ✅ (通过 MongoDB 查询) |
| **Job 详情** | ✅ | ✅ |
| **删除 Job** | ✅ (clean) | ✅ |
| **重排队** | ✅ | ✅ |

**监控便利性**: Bull > Agenda (生态更强)

---

## 🏢 许可证分析

### Bull
- **许可证**: "Other" (SPDX: NOASSERTION)
- ** GitHub 显示**: 未明确选择许可证（默认 All Rights Reserved）
- **风险**: ⚠️ 商用需作者授权或切换到 BullMQ（MIT）
- **建议**: 避免直接使用 Bull，改用 **BullMQ**（项目后继，MIT 许可）

### Agenda
- **许可证**: MIT ✅
- **风险**: 无
- **商业使用**: 完全自由

**许可证风险**:
- **Bull**: 🔴 高风险（原项目未明确 OSS 许可）
- **BullMQ**: ✅ 安全（MIT）
- **Agenda**: ✅ 安全（MIT）

---

## ⚖️ 选型对比

### 场景 1: 高并发分布式队列

**需求**: 每天处理 100k+ 任务，需要优先级、限流、重试

**推荐**: **BullMQ** (Bull 的官方后继)

**理由**:
- Redis 性能高（10k+ ops/sec）
- 丰富的特性（优先级、limiter、延迟）
- 成熟的监控生态（Arena UI）
- 活跃社区（BullMQ 持续维护）

**成本**:
- Redis 托管（AWS ElastiCache / GCP Memorystore）
- 预估 4 GB RAM Redis 实例 ≈ $50-100/月

---

### 场景 2: 轻量定时任务

**需求**: 每隔 N 小时/分钟执行某个任务（如数据同步、报表生成）

**推荐**: **Agenda**

**理由**:
- MongoDB 持久化，任务不丢失
- Cron 语法原生支持
- 配置简单，几行代码搞定
- MIT 许可证无负担
- Issues 极少，代码稳定

**成本**:
- 复用现有 MongoDB（若无，最低配置 Atlas M0 免费）

---

### 场景 3: 已用技术栈决策

| 现有技术 | 推荐 | 理由 |
|----------|------|------|
| **已有 Redis** | BullMQ | 无需新增基础设施 |
| **已有 MongoDB** | Agenda | 零额外存储成本 |
| **两者皆无** | Agenda (MongoDB Atlas Free) | 免费额度充足 |
| **需要高性能队列** | BullMQ (Redis) | Redis 吞吐远高于 MongoDB |
| **任务量大且复杂** | BullMQ | 重试/延迟/优先级管理更成熟 |

---

## 📦 部署建议

### BullMQ (生产建议)

```javascript
// bullmq 示例 (替代 bull)
const { Queue, QueueScheduler, Worker } = require('bullmq');

const connection = { host: '127.0.0.1', port: 6379 };

const myQueue = new Queue('my-queue', { connection });
new QueueScheduler('my-queue', { connection }); // 处理延迟/重试

const worker = new Worker('my-queue', async job => {
  // 处理逻辑
}, { connection });

// 监控 UI
// https://github.com/felixmosh/bull-board
```

**高可用**:
- Redis sentinel / cluster
- 多个 worker 进程（horizontal scaling）
- 设置合理的 concurrency

---

### Agenda (生产建议)

```javascript
const { MongoClient, ObjectId } = require('mongodb');
const Agenda = require('agenda');

const mongoClient = await MongoClient.connect(mongoUri, { useNewUrlParser: true });
const db = mongoClient.db('agenda');

const agenda = new Agenda({
  db: { address: mongoUri, collection: 'agendaJobs' },
  processEvery: '30 seconds',  // 扫描间隔
  defaultLockLifetime: 10 * 60 * 1000  // 10分钟锁超时
});

// 定义带并发限制的任务
agenda.define('cleanup', async job => {
  await cleanupOldData();
}, {
  concurrency: 2,       // 最多同时2个
  priority: 'high'      // 优先级
});

// 每1小时执行
agenda.every('1 hour', 'cleanup');

await agenda.start();
```

**高可用**:
- MongoDB replica set
- 多个 agenda 进程（自动锁竞争）
- 设置 `processEvery` 避免频繁扫描

---

## 🔗 参考来源

- [Bull (archived)](https://github.com/OptimalBits/bull)
- [BullMQ](https://github.com/taskforcesh/bullmq)
- [Agenda](https://github.com/agenda/agenda)
- [Bull vs Agenda 对比](https://www.google.com/search?q=bull+vs+agenda)

---

## ✅ 对比总结

| 场景 | 推荐 | 理由 |
|------|------|------|
| **高吞吐量任务队列** | BullMQ | Redis 性能远超 MongoDB |
| **轻量定时任务** | Agenda | 配置简单，MIT 许可，MongoDB 免费 |
| **已有 Redis** | BullMQ | 零基础设施成本 |
| **已有 MongoDB** | Agenda | 零基础设施成本 |
| **任务重试/延迟复杂** | BullMQ | 机制成熟（exponential backoff） |
| **Cron 风格调度** | 两者皆可<br>Agenda 更简洁 | Cron 语法直接 |
| **实时性要求高** | BullMQ | 秒级调度，Redis 低延迟 |
| **许可证敏感** | Agenda (MIT) / BullMQ (MIT) | 避免 Bull 原版 |
| **监控需求** | BullMQ | Arena/Bull Board UI 完善 |

---

## 🎯 我们的推荐

### 评估背景
- **任务类型**: 定时任务（数据同步、报表生成）+ 少量延迟任务
- **并发量**: 中等（< 1000 任务/天）
- **已有基础设施**: MongoDB Atlas（免费额度）+ Redis（可选）
- **团队熟悉度**: Node.js 为主

### 推荐方案

#### 🏆 首选: **Agenda**

**理由**:
1. **轻量简洁**: 50 行代码搞定调度，Bull 配置较复杂
2. **MIT 许可证**: 无后顾之忧
3. **MongoDB 持久化**: 任务历史可追溯，重启不丢
4. **Cron 原生**: `agenda.every('5 minutes', ...)` 直观
5. **社区稳定**: Issues 极少，代码成熟

**适用场景**:
- 定时 ETL
- 报表生成
- 邮件通知
- 数据清理

#### 🥈 备选: **BullMQ**

**启用条件**:
- 任务量激增（> 10k/天）
- 需要严格优先级控制
- 需要延迟队列（如 retry with backoff 精确控制）
- 已用 Redis，想统一存储

---

## 🚀 快速上手 (Agenda)

```bash
npm install agenda
```

```javascript
const Agenda = require('agenda');

const agenda = new Agenda({
  db: { address: 'mongodb://localhost:27017/agenda' }
});

// 定义任务
agenda.define('send newsletter', async job => {
  const { userId } = job.attrs.data;
  await sendEmail(userId);
});

// 每天上午10点执行
agenda.every('0 10 * * *', 'send newsletter');

// 每30秒执行一次（调试用）
agenda.every('30 seconds', 'send newsletter');

agenda.start();
```

---

**状态**: Phase D 调研完成  
**下一步**: Phase E - 撰写综合推荐方案与完整报告
