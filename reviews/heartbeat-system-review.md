# 代码审查报告：心跳系统 (heartbeat.js)

**审查人**: xiaoyu  
**审查时间**: 2026-03-17  
**审查对象**: `core/handlers/heartbeat.js`

---

## 📋 审查要点

1. 心跳上报逻辑
2. Agent 状态管理
3. 超时处理
4. 并发安全
5. 错误处理

---

## ✅ 优点

### 1. 功能丰富
- 基础心跳上报 ✅
- 在线状态管理 ✅
- 超时清理机制 ✅
- 心跳日志记录 ✅
- 智能空闲判定 ✅
- 自动认领任务 ✅
- 状态变化通知 ✅

### 2. 代码结构清晰
- 函数职责单一 ✅
- 注释详细 ✅
- 配置驱动 ✅

### 3. 智能特性
- 低负载自动判定空闲 ✅
- 任务完成自动切换 idle ✅
- 空闲自动认领任务 ✅
- 通知去重机制 ✅

---

## 🔴 严重问题

### 问题 1：模块导出重复

**现状**:
```javascript
// 第 235 行
module.exports = {
  handleHeartbeat,
  getOnlineAgents,
  getAgentStatus,
  cleanupOfflineAgents,
  onlineAgents
}

// 第 289 行
module.exports.registerAgent = registerAgent

// 第 393 行
module.exports.registerAgent = registerAgent  // 重复导出！
```

**风险**: 
- 代码冗余，维护困难
- 可能存在两个不同版本的 `registerAgent` 函数

**建议**:
```javascript
// 统一在文件末尾导出一次
module.exports = {
  handleHeartbeat,
  getOnlineAgents,
  getAgentStatus,
  cleanupOfflineAgents,
  registerAgent,
  onlineAgents
}
```

**优先级**: 🔴 高

---

### 问题 2：registerAgent 函数重复定义

**现状**:
- 第 244-288 行：第一个 `registerAgent` 版本（临时 TOKEN）
- 第 292-392 行：第二个 `registerAgent` 版本（正式 TOKEN + .env 写入）

**风险**: 
- 第二个定义覆盖第一个，代码混乱
- 维护时可能修改错误的版本
- 代码审查困难

**建议**: 
- 删除第一个版本（临时 TOKEN）
- 保留第二个版本（完整功能）
- 或者合并两个版本的优点

**优先级**: 🔴 高

---

### 问题 3：内存泄漏风险

**现状**:
```javascript
// 定时清理，每 1 分钟
setInterval(cleanupOfflineAgents, 60000)
```

**风险**: 
- 与 stats.js 相同的问题
- 模块多次加载会创建多个定时器

**建议**:
```javascript
// 使用单次 setTimeout 递归
function scheduleCleanup() {
  setTimeout(() => {
    cleanupOfflineAgents()
    scheduleCleanup()
  }, 60000)
}
scheduleCleanup()
```

**优先级**: 🟡 中

---

### 问题 4：文件路径硬编码

**现状**:
```javascript
const envPath = path.join(__dirname, '../../.env')
```

**风险**: 
- 依赖固定的目录结构
- 如果模块移动，路径失效

**建议**:
```javascript
// 使用 config 中的路径配置
const envPath = process.env.ENV_FILE_PATH || 
                path.join(__dirname, '../../.env')
```

**优先级**: 🟢 低

---

### 问题 5：.env 文件写入无锁

**现状**:
```javascript
fs.writeFileSync(envPath, envContent, 'utf-8')
```

**风险**: 
- 多个 Agent 同时注册时可能冲突
- 文件内容可能损坏

**建议**:
```javascript
// 添加文件锁
const lock = require('proper-lockfile')

async function writeEnvToken(envPath, envVarName, token) {
  const release = await lock.lock(envPath)
  try {
    // 读写操作
  } finally {
    await release()
  }
}
```

**优先级**: 🟡 中

---

### 问题 6：错误处理不完整

**现状**:
```javascript
try {
  // .env 写入
} catch (error) {
  console.error('[REGISTER] ⚠️ 写入 .env 失败:', error.message)
  // 没有返回错误，仍然返回成功
}
```

**风险**: 
- 写入失败但返回成功，调用方误以为成功
- TOKEN 实际未保存

**建议**:
```javascript
catch (error) {
  console.error('[REGISTER] ⚠️ 写入 .env 失败:', error.message)
  // 返回部分成功标志
  return {
    success: true,
    message: '注册成功，但写入 .env 失败',
    warning: error.message,
    // ...
  }
}
```

**优先级**: 🟡 中

---

### 问题 7：安全漏洞 - 口令硬编码

**现状**:
```javascript
const VALID_PASSPHRASE = process.env.REGISTRATION_PASSPHRASE || 'YTISMYB@SS'
```

**风险**: 
- 默认口令硬编码在代码中
- 攻击者可以轻易获取
- 如果未配置环境变量，使用弱口令

**建议**:
```javascript
// 强制要求配置，不提供默认值
const VALID_PASSPHRASE = process.env.REGISTRATION_PASSPHRASE
if (!VALID_PASSPHRASE) {
  throw new Error('必须配置 REGISTRATION_PASSPHRASE 环境变量')
}
```

**优先级**: 🔴 高

---

### 问题 8：通知功能未实现

**现状**:
```javascript
// TODO: 调用飞书通知 API
// sendFeishuNotification(agent_id, finalStatus, current_task, shouldNotify.reason)
```

**风险**: 
- 功能未完成
- 状态变化无法及时通知

**建议**: 
- 实现飞书通知集成
- 或者移除相关代码

**优先级**: 🟡 中

---

## 📊 审查评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 心跳上报逻辑 | 8/10 | 功能完整，智能判定优秀 |
| Agent 状态管理 | 7/10 | Map 存储合理，但缺持久化 |
| 超时处理 | 7/10 | 有清理机制，但定时器有泄漏 |
| 并发安全 | 4/10 | .env 写入无锁，竞态风险 |
| 错误处理 | 5/10 | 基础处理，但部分场景不完整 |
| 安全性 | 3/10 | 口令硬编码，严重安全隐患 |

**总体评分**: **5.7/10** ⚠️

---

## 📌 改进建议优先级

### 🔴 高优先级（必须修复）
1. **删除重复代码** - 合并两个 `registerAgent` 版本
2. **修复安全漏洞** - 强制配置注册口令，移除默认值
3. **统一模块导出** - 在文件末尾统一导出

### 🟡 中优先级（推荐修复）
4. **修复定时器泄漏** - 改用递归 setTimeout
5. **完善错误处理** - .env 写入失败应返回警告
6. **实现通知功能** - 集成飞书 API 或移除 TODO

### 🟢 低优先级（可选优化）
7. **添加文件锁** - .env 写入并发安全
8. **配置化路径** - 避免硬编码

---

## 🔍 代码质量对比

| 模块 | 总体评分 | 关键问题 |
|------|----------|----------|
| stats.js | 5.8/10 | 并发安全、内存泄漏 |
| heartbeat.js | 5.7/10 | 重复代码、安全漏洞 |
| task.js (删除 API) | N/A | 功能未实现 |

**结论**: 心跳系统代码质量与统计系统相当，都需要优先修复高优先级问题。

---

## 📅 下一步

1. Lucy 优先修复重复代码和安全漏洞
2. 实现飞书通知集成
3. 添加单元测试覆盖边界情况

---

*审查完成时间：2026-03-17 17:55*
