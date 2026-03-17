# 代码审查报告：删除任务 API

**审查人**: xiaoyu  
**审查时间**: 2026-03-17  
**任务 ID**: task_1773728536888_vw4hsr  
**优先级**: P1

---

## 📋 审查对象

- `core/handlers/task.js` - 任务处理模块
- `core/index.js` - HTTP 路由

---

## 🔍 审查发现

### ❌ 问题 1：删除任务 API 未实现

**现状**:
- `task.js` 中**没有** `deleteTask` 函数
- `index.js` 中**没有** `/api/tasks/:id` 的 DELETE 路由
- 任务模块功能列表中包含"任务创建/读取/更新/删除"，但删除功能缺失

**代码位置**:
```javascript
// task.js 现有函数列表
- createTask()      ✅
- getTasks()        ✅
- claimTask()       ✅
- submitTask()      ✅
- pauseTask()       ✅
- resumeTask()      ✅
- updateTaskStatus()✅
- deleteTask()      ❌ 缺失！
```

---

## ⚠️ 问题 2：缺少删除相关的安全考虑

如果实现删除功能，需要考虑以下安全问题：

### 2.1 权限检查
- ❓ 谁可以删除任务？
  - 仅创建者 (lucy)?
  - 任务负责人 (assignee)?
  - 所有人？

### 2.2 状态检查
- ❓ 什么状态的任务可以删除？
  - 仅 pending 任务？
  - working 状态的任务是否需要先暂停？
  - completed 任务是否允许删除（还是只能归档）？

### 2.3 级联处理
- ❓ 删除任务时的关联处理：
  - 已暂停的任务如何处理？
  - 有 callback 的任务如何处理？
  - 是否需要通知相关 Agent？

### 2.4 软删除 vs 硬删除
- ❓ 删除策略：
  - 软删除（标记 deleted 字段）- 推荐 ✅
  - 硬删除（从文件移除）- 有风险 ⚠️

---

## 📝 建议实现方案

### 推荐 API 设计

```javascript
// DELETE /api/tasks/:id
// 请求头：x-claw-token: <agent_token>

// 响应示例
{
  "success": true,
  "task": {
    "id": "task_xxx",
    "status": "deleted",
    "deleted_at": 1773731800000,
    "deleted_by": "xiaoyu"
  }
}
```

### 推荐实现逻辑

```javascript
async function deleteTask(taskId, agentId, agentRole) {
  const tasks = readTasks()
  const task = tasks.find(t => t.id === taskId)
  
  if (!task) {
    return { success: false, error: '任务不存在' }
  }
  
  // 权限检查：仅创建者或任务负责人可删除
  if (agentRole !== 'leader' && task.assignee !== agentId) {
    return { success: false, error: '权限不足' }
  }
  
  // 状态检查：working 状态的任务不能直接删除
  if (task.status === 'working') {
    return { 
      success: false, 
      error: '任务正在进行中，请先暂停或提交' 
    }
  }
  
  // 软删除：标记删除状态
  task.status = 'deleted'
  task.deleted_at = Date.now()
  task.deleted_by = agentId
  
  writeTasks(tasks)
  
  // 移动到已完成任务（可选）
  // 或保留在 active.json 但标记为 deleted
  
  return { success: true, task }
}
```

---

## ✅ 审查结论

| 审查项 | 状态 | 说明 |
|--------|------|------|
| 权限检查 | ❌ 未实现 | 需要实现 |
| 状态检查 | ❌ 未实现 | 需要实现 |
| 错误处理 | ❌ 未实现 | 需要实现 |
| 日志记录 | ❌ 未实现 | 需要实现 |
| 安全漏洞 | ⚠️ 待评估 | 功能未实现，无法评估 |

---

## 📌 改进建议

### 优先级 1（必须）
1. **实现 deleteTask 函数** - 核心功能缺失
2. **添加 DELETE 路由** - `/api/tasks/:id`
3. **权限检查** - 仅 leader 或 assignee 可删除
4. **状态检查** - working 状态不可删除

### 优先级 2（推荐）
5. **软删除** - 标记 deleted 字段而非物理删除
6. **删除日志** - 记录删除者、时间、原因
7. **Webhook 通知** - 删除时触发通知

### 优先级 3（可选）
8. **批量删除** - 支持批量删除已完成任务
9. **恢复功能** - 支持恢复误删的任务
10. **删除原因** - 可选填写删除原因

---

## 🎯 代码质量评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 功能完整性 | 0/10 | 功能未实现 |
| 安全性 | N/A | 无法评估 |
| 可维护性 | N/A | 无法评估 |
| 性能 | N/A | 无法评估 |

**总体评分**: **N/A** - 功能未实现，无法评分

---

## 📅 下一步

1. Lucy 需要实现 `deleteTask` 函数
2. 添加 DELETE 路由到 `index.js`
3. 编写单元测试
4. 重新审查

---

*审查完成时间：2026-03-17 15:30*
