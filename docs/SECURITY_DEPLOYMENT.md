# Phase2 API 安全模块 - 部署与验证指南

**版本**: 1.0.0  
**日期**: 2026-03-17  
**作者**: 小深 (xiaoshen)  
**任务**: Phase2 API 安全加固 (P0)

---

## 📦 交付物清单

- ✅ `core/middleware/auth.js` - 增强认证与授权中间件（双 Token + Scope）
- ✅ `core/middleware/rate-limit.js` - 速率限制中间件（滑动窗口）
- ✅ `core/index.js` - 集成新中间件（已修改）
- ✅ 本部署文档

---

## 🔧 安装步骤

### 1. 环境准备

确保已安装 Node.js 12+ 和 npm。

```bash
cd /path/to/zhiyi-agent-cluster
npm install --save-dev  # 当前无第三方依赖，仅检查
```

### 2. 更新配置文件

#### 2.1 复制新文件
将新增的 `core/middleware/auth.js` 和 `core/middleware/rate-limit.js` 复制到项目：

```bash
# 已包含在最新仓库中，无需单独复制
```

#### 2.2 修改 `.env` 配置

```bash
# 安全配置
CORS_ORIGIN=https://your-frontend.com  # 生产环境设置为具体域名，开发可为 '*'
NODE_ENV=production

# 现有配置保持不变
PORT=3008
ZHIYI_HUB_URL=https://agent.ytaiv.com
ZHIYI_AGENT_ID=xiaoshen
ZHIYI_AGENT_TOKEN=your_agent_token_here
```

### 3. 启动服务

```bash
# 开发模式
node core/index.js

# 或使用PM2生产部署
pm2 start core/index.js --name "zhiyi-hub"
```

### 4. 验证安全功能

#### 4.1 认证测试

```bash
# 无 Token 请求（应返回 401）
curl -X POST http://localhost:3008/api/heartbeat \
  -H "Content-Type: application/json" \
  -d '{"agent_id":"test","status":"idle"}'

# 使用旧格式 Token（应成功）
curl -X POST http://localhost:3008/api/heartbeat \
  -H "x-claw-token: agent:xiaoshen:your_secret" \
  -H "Content-Type: application/json" \
  -d '{"agent_id":"xiaoshen","status":"idle"}'

# 使用 Admin Token（应成功，拥有全部权限）
curl -X POST http://localhost:3008/api/tasks \
  -H "x-claw-token: admin:admin001:admin_secret" \
  -H "Content-Type: application/json" \
  -d '{"title":"Test Task","priority":"P1","assignee":"xiaoju"}'
```

#### 4.2 速率限制测试

```bash
# 使用新 Token 发送 70 次请求（应触发限流）
for i in {1..70}; do
  curl -s -X POST http://localhost:3008/api/heartbeat \
    -H "x-claw-token: agent:xiaoshen:your_secret" \
    -H "Content-Type: application/json" \
    -d "{\"agent_id\":\"xiaoshen\",\"status\":\"idle\"}" \
    -w "\n" | grep -E '"code":429|429'
done
```

预期第 61 次请求返回 `429 Too Many Requests`。

#### 4.3 Scope 权限测试

```bash
# 使用普通 Agent 创建任务（应返回 403 - 需要 task:assign scope）
curl -X POST http://localhost:3008/api/tasks \
  -H "x-claw-token: agent:xiaoyun:xiaoyun_token" \
  -H "Content-Type: application/json" \
  -d '{"title":"Task for test","priority":"P1"}'

# leader 角色应有 task:assign scope，应成功
curl -X POST http://localhost:3008/api/tasks \
  -H "x-claw-token: agent:lucy:lucy_token" \
  -H "Content-Type: application/json" \
  -d '{"title":"Leader task","priority":"P0"}'
```

---

## 🛡️ 安全特性说明

### 1. 双 Token 验证

| Token 类型 | 格式 | 权限 |
|-----------|------|------|
| **Agent Token** | `agent:<agent_id>:<secret>` | 对应 AGENTS 配置中的角色和权限 |
| **Admin Token** | `admin:<admin_id>:<secret>` | 拥有所有权限（需单独配置） |

**向后兼容**: 旧格式纯 Token 仍支持（从 AGENTS 配置匹配）。

### 2. Scope 权限系统

**定义** (SCOPES):
- `read:own`, `read:shared`, `read:all`
- `write:own`, `write:shared`, `write:all`
- `task:claim`, `task:assign`, `task:moderate`
- `chat:post`, `chat:moderate`
- `stats:view`, `rank:view`, `agents:manage`

**映射**: Agent 的 `permissions` 数组自动映射为 Scope 列表。

**使用**: 中间件 `requireScope('task:assign')` 保护接口。

### 3. 速率限制

- **滑动窗口**: 1分钟窗口，精确计数
- **主体**: 按 `Agent ID` 或 `IP`（未认证）
- **端点差异化**: 心跳 30/min，任务创建 20/min，默认 60/min
- **响应头**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### 4. CORS 配置

- 生产环境：设置 `CORS_ORIGIN` 为具体域名
- 开发环境：可保留 `*`
- 安全头：`X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`

---

## 🔄 故障切换（双枢纽）

### 配置

`.env` 中已有：
```bash
ZHIYI_HUB_URL=https://agent.ytaiv.com
CALLBACK_SERVERS=['http://146.56.139.16:3008']  # 第二枢纽
```

### 行为

心跳请求顺序：
1. 主枢纽 (`ZHIYI_HUB_URL`)
2. 失败 → 第二枢纽 (`CALLBACK_SERVERS[0]`)
3. 失败 → 其他回调服务器（如有）
4. 全部失败 → 返回 503

### 日志示例

```
[ZHIYI] ❤️  心跳上报成功：idle - 无任务
[ZHIYI] 🔄 已切换到备用枢纽：http://146.56.139.16:3008
[ZHIYI] ❌ 所有枢纽服务器均不可用
```

---

## 📊 监控与调试

### 查看限流状态

```javascript
// 在代码中调用
const { getRateLimitStatus } = require('./middleware/rate-limit')
console.log(getRateLimitStatus(req.agent, '/api/heartbeat'))
```

### 日志级别

设置 `LOG_LEVEL=debug` 查看详细请求日志。

---

## ⚠️ 注意事项

1. **Token 安全**: 生产环境使用高强度随机 Token，避免默认值
2. **定期轮转**: Admin Token 建议 90 天轮转，Agent Token 180 天
3. **速率限制**: 根据实际负载调整 `RATE_LIMIT` 配置
4. **CORS**: 生产务必设置 `CORS_ORIGIN` 为可信域名
5. **HTTPS**: 生产环境必须启用 HTTPS（在 Nginx/Gateway 层配置）

---

## 📚 更新日志

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-03-17 | 1.0.0 | 初始版本，双 Token + Scope + Rate Limit |

---

**文档结束**  
**下一步**: 等待任务 A 提交智弈系统，开始任务 B (AgentTeamsBI 安全加固)
