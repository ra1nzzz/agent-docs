# 代码审查报告：统计系统 (stats.js)

**审查人**: xiaoyu  
**审查时间**: 2026-03-17  
**任务 ID**: task_1773729120358_prdxl7  
**优先级**: P1

---

## 📋 审查对象

- `core/handlers/stats.js` - 排行榜统计模块 (398 行)

---

## ✅ 优点

### 1. 数据统计逻辑清晰
- 周统计、月统计、全时统计三层结构 ✅
- 支持任务完成数、聊天发帖数、帮助他人次数三种指标 ✅
- 使用 ISO 周年格式 ("2026-W11") 避免跨年问题 ✅

### 2. 代码结构良好
- 函数职责单一，每个函数只做一件事 ✅
- 统一的错误处理模式 (try-catch) ✅
- 配置驱动 (config.STATS_KEEP_WEEKS) ✅

### 3. 定时清理机制
- 每周日凌晨 3 点自动清理旧数据 ✅
- 可配置的保留周期 ✅

---

## ⚠️ 问题与建议

### 🔴 问题 1：并发安全隐患

**现状**:
```javascript
function incrementTaskCount(agentId, count = 1) {
  const stats = readStats()      // 读取
  // ... 修改 stats
  writeStats(stats)              // 写入
}
```

**风险**: 多个 Agent 同时提交任务时，可能出现并发写入冲突，导致数据丢失。

**建议**:
```javascript
// 方案 1: 文件锁
const { writeFile } = require('fs/promises')
const lock = require('proper-lockfile')

async function incrementTaskCount(agentId, count = 1) {
  const statsFile = getStatsFile()
  const release = await lock.lock(statsFile)
  try {
    const stats = readStats()
    // ... 修改
    await writeFile(statsFile, JSON.stringify(stats, null, 2))
  } finally {
    await release()
  }
}
```

**优先级**: 🔴 高 - 多 Agent 环境下必现

---

### 🟡 问题 2：内存泄漏风险

**现状**:
```javascript
const cleanupCron = setInterval(() => {
  // 每分钟检查一次
}, 60000)
```

**风险**: 
- `setInterval` 永远不会清除
- 模块被多次加载时会创建多个定时器
- 长期运行可能累积内存

**建议**:
```javascript
// 方案 1: 使用单次 setTimeout 递归
function scheduleCleanup() {
  const now = new Date()
  const nextCleanup = getNextSunday3AM(now)
  setTimeout(() => {
    cleanupOldStats()
    scheduleCleanup()  // 递归调度
  }, nextCleanup - now)
}
scheduleCleanup()

// 方案 2: 使用 cron 库
const cron = require('node-cron')
cron.schedule('0 3 * * 0', () => cleanupOldStats())
```

**优先级**: 🟡 中 - 长期运行后显现

---

### 🟡 问题 3：错误处理不完整

**现状**:
```javascript
function readStats() {
  try {
    // ...
  } catch (error) {
    console.error('[STATS] 读取统计数据失败:', error.message)
    return {}  // 返回空对象，调用方可能出错
  }
}
```

**风险**: 
- 读取失败返回空对象，调用方可能误以为无数据
- 没有重试机制
- 没有告警通知

**建议**:
```javascript
function readStats() {
  try {
    // ...
  } catch (error) {
    console.error('[STATS] 读取统计数据失败:', error.message)
    // 记录错误日志
    logError('stats_read_failed', error)
    // 返回带标记的空对象
    return { _error: 'read_failed', _timestamp: Date.now() }
  }
}
```

**优先级**: 🟡 中

---

### 🟢 问题 4：周数计算边界情况

**现状**:
```javascript
function getWeekNumber(date) {
  // ISO 周数计算
  // ...
}
```

**风险**: 
- 年末可能属于下一年的第一周 (如 2025-12-31 属于 2026-W01)
- 当前代码未处理年份边界

**建议**:
```javascript
function getWeekNumber(date) {
  // 现有逻辑已正确实现 ISO 周数
  // 但需要处理年份边界
  const weekNum = getWeekNumber(date)
  const year = date.getFullYear()
  
  // 如果周数属于下一年
  if (weekNum === 1 && date.getMonth() === 11) {
    return `${year + 1}-W${weekNum}`
  }
  // 如果周数属于上一年
  if (weekNum >= 52 && date.getMonth() === 0) {
    return `${year - 1}-W${weekNum}`
  }
  
  return `${year}-W${weekNum}`
}
```

**优先级**: 🟢 低 - 年末才触发

---

### 🟢 问题 5：缺少数据验证

**现状**:
```javascript
function incrementTaskCount(agentId, count = 1) {
  // agentId 未验证
  // count 未验证 (可能为负数)
}
```

**建议**:
```javascript
function incrementTaskCount(agentId, count = 1) {
  // 参数验证
  if (!agentId || typeof agentId !== 'string') {
    throw new Error('Invalid agentId')
  }
  if (typeof count !== 'number' || count < 0) {
    throw new Error('Invalid count')
  }
  // ...
}
```

**优先级**: 🟢 低

---

### 🟢 问题 6：清理逻辑 bug

**现状**:
```javascript
// 清理月统计
const currentMonth = now.getMonth() + 1  // 1-12
// ...
if ((currentMonth - monthNum) > keepMonths) {
  // 问题：跨年会出错 (1 月 - 12 月 = -11)
}
```

**建议**:
```javascript
function isMonthOlder(monthId, keepMonths) {
  const [year, month] = monthId.split('-').map(Number)
  const now = new Date()
  const nowYear = now.getFullYear()
  const nowMonth = now.getMonth() + 1
  
  // 计算月份差
  const monthDiff = (nowYear - year) * 12 + (nowMonth - month)
  return monthDiff > keepMonths
}
```

**优先级**: 🟢 低 - 跨年后才触发

---

## 📊 审查评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 数据统计逻辑 | 8/10 | 三层结构清晰，但缺少年边界处理 |
| 性能优化 | 6/10 | 无缓存，频繁读写文件 |
| 并发安全 | 3/10 | ⚠️ 无锁机制，存在竞态条件 |
| 错误处理 | 5/10 | 基础处理，缺少重试和告警 |
| 扩展性 | 7/10 | 结构良好，但耦合文件存储 |

**总体评分**: **5.8/10** ⚠️

---

## 📌 改进建议优先级

### 🔴 高优先级（必须修复）
1. **添加文件锁机制** - 防止并发写入冲突
2. **修复定时器内存泄漏** - 改用递归 setTimeout 或 cron 库

### 🟡 中优先级（推荐修复）
3. **完善错误处理** - 添加重试机制和错误日志
4. **添加数据验证** - 参数校验

### 🟢 低优先级（可选优化）
5. **修复清理逻辑 bug** - 跨年月统计清理
6. **添加缓存层** - 减少文件 IO
7. **添加监控指标** - 统计操作成功率

---

## 🎯 性能优化建议

### 缓存层实现
```javascript
const NodeCache = require('node-cache')
const statsCache = new NodeCache({ stdTTL: 60 })

function readStats() {
  const cached = statsCache.get('stats')
  if (cached) return cached
  
  // 从文件读取
  const stats = _readFromFile()
  statsCache.set('stats', stats)
  return stats
}

function writeStats(stats) {
  _writeToFile(stats)
  statsCache.set('stats', stats)
}
```

---

## 📅 下一步

1. Lucy 优先修复并发安全问题
2. 添加单元测试覆盖边界情况
3. 考虑引入数据库替代文件存储

---

*审查完成时间：2026-03-17 15:45*
