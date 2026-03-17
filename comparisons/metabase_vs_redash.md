# Metabase vs Redash - BI 可视化工具对比

**调研日期**: 2026-03-17  
**调研人**: 小深 (xiaoshen)  
**领域**: Business Intelligence & Embedded Analytics

---

## 📊 核心数据对比

| 维度 | Metabase | Redash |
|------|----------|--------|
| **GitHub Stars** | 46,421 ⭐ | 28,280 ⭐ |
| **Forks** | 6,307 | 4,564 |
| **Issues** | 3,984 (开放) | 743 (开放) |
| **语言栈** | Clojure + React | Python + React |
| **许可证** | 自定义 (AGPL 衍生) | BSD-2-Clause ✅ |
| **最新活跃** | 高频 (2026-03-17) | 中频 (2026-03-13) |
| **官网** | metabase.com | redash.io |

---

## 🎯 核心定位

### Metabase
> "The easy-to-use open source Business Intelligence and Embedded Analytics tool that lets everyone work with data"

**关键词**: 易用性、嵌入式分析、面向全员

### Redash
> "Make Your Company Data Driven. Connect to any data source, easily visualize, dashboard and share your data."

**关键词**: 数据连接、可视化、共享

---

## 🔧 架构与部署

### Metabase
- **架构**: Clojure 后端 + React 前端
- **部署**: JAR 包或 Docker
- **数据库**: 内置 H2（生产推荐 PostgreSQL）
- **复杂度**: 中等（JVM 环境）

### Redash
- **架构**: Python (Flask) 后端 + React 前端
- **部署**: Docker Compose（推荐）或源码
- **数据库**: PostgreSQL
- **复杂度**: 较低（容器化部署）

**部署便捷性**: Redash > Metabase (Docker 开箱即用)

---

## 📈 核心功能对比

### 数据源连接
| 功能 | Metabase | Redash |
|------|----------|--------|
| **支持数量** | 40+ 数据库 | 50+ 数据源 |
| **常见数据库** | ✅ MySQL/Postgres/SQL Server<br>✅ MongoDB<br>✅ ClickHouse<br>✅ Druid | ✅ MySQL/Postgres/SQL Server<br>✅ MongoDB<br>✅ ClickHouse<br>✅ BigQuery/Redshift/Snowflake |
| **云服务** | 有限 | 广泛（AWS/Google Cloud） |
| **自定义驱动** | 支持 JDBC | 支持 Python 驱动 |

**结论**: Redash 在云数据仓库支持上更全面。

---

### 查询与可视化

#### Metabase
- **查询方式**:
  - 可视化查询构建器（拖拽式，SQL 零基础可用）
  - 原生 SQL 编辑器（高级用户）
  - 问题 (Questions) 概念
- **可视化类型**: 14+ 图表（柱状图、折线图、饼图、地图、指标卡等）
- **仪表板**: 可交互、支持筛选器、自动刷新
- **特色功能**:
  - 🔍 **数据轮廓** (Data Profile): 自动分析字段分布
  - 📊 **简洁 UI**: 直观易懂，学习曲线平缓
  - 🔄 **Embedded Analytics**: API 嵌入第三方应用

#### Redash
- **查询方式**:
  - 纯 SQL 编辑器（Query 概念）
  - 可视化查询辅助（较少）
- **可视化类型**: 丰富图表库 + 自定义可视化
- **仪表板**: 支持小组件、参数、联动
- **特色功能**:
  - 🔁 **查询结果缓存**: 提升性能
  - 🤝 **协作功能**: 查询复用、版本历史
  - 🔌 **API 优先**: 完善的 REST API

**易用性**: Metabase > Redash (可视化构建器降低门槛)  
**灵活性**: Redash > Metabase (SQL 自由度更高)

---

### 权限与协作

| 功能 | Metabase | Redash |
|------|----------|--------|
| **用户角色** | 管理员/编辑者/查看者/来宾 | 管理员/编辑者/查看者 |
| **组权限** | ✅ 支持 | ✅ 支持 |
| **数据行级权限** | ✅ (基于字段过滤) | ❌ (需自定义查询) |
| **审计日志** | ✅ | ✅ |
| **SSO** | ✅ (SAML, LDAP, JWT) | ✅ (LDAP, Google OAuth) |

**企业级功能**: Metabase 略胜一筹（行级权限更完善）

---

### 告警与订阅

- **Metabase**: 支持仪表板定时邮件 + Slack 通知（需配置）
- **Redash**: 支持查询结果邮件 + Webhook + Slack

功能相似，Redash 的 Webhook 更灵活。

---

## 🏢 许可证分析

### Metabase
- **许可证**: AGPL v3 的衍生（"Other"）
- **限制**:
  - 开源版本功能完整
  - 云托管版本（Metabase Cloud）提供额外功能
  - 企业版功能需购买
- **商业使用**: ✅ 允许

### Redash
- **许可证**: BSD-2-Clause ✅
- **限制**: 无
- **商业使用**: ✅ 完全自由

**许可证风险**:
- **Metabase**: AGPL 传染性需注意（SaaS 场景可能需开源修改）
- **Redash**: 商业友好，无负担

---

## 🎪 社区与生态

| 指标 | Metabase | Redash |
|------|----------|--------|
| GitHub Stars | 46.4k | 28.3k |
| Forks 比例 | 13.6% | 16.1% |
| 贡献者 | 活跃社区 | 较活跃 |
| 文档质量 | 详细，有中文 | 详细，英文为主 |
| 第三方教程 | 丰富 | 丰富 |
| 插件生态 | 中等（数据源驱动） | 丰富（可视化插件） |

---

## ⚖️ 对比总结

| 场景 | 推荐 | 理由 |
|------|------|------|
| **全员自助分析** | Metabase | 可视化查询构建器，SQL 零基础也能用 |
| **数据分析师主导** | Redash | SQL 灵活，API 强大，适合技术团队 |
| **云原生部署** | Redash | Docker Compose 一键部署，K8s 友好 |
| **嵌入式分析** | Metabase | 官方提供 Embedded Analytics SDK |
| **多云/混合云** | Redash | 数据源支持更广（BigQuery/Redshift/Snowflake） |
| **许可证敏感** | Redash | BSD 无传染性，商业使用无顾虑 |
| **行级权限严格** | Metabase | 内置 Row Level Permissions |
| **小团队快速启动** | 两者皆可 | Redash 略快，Metabase 更易用 |

---

## 💡 我们的推荐

### 评估背景
- **团队规模**: 小团队（5-10 人）
- **技术背景**: 有数据分析能力，非全员需深度使用
- **部署环境**: 自托管（Docker）
- **主要需求**: 业务指标监控、报表生成、数据探索

### 推荐方案

#### 🏆 首选: **Redash**

**理由**:
1. **部署简单**: Docker Compose 5 分钟搞定，Metabase JVM 稍重
2. **许可证干净**: BSD-2-Clause，商业无后顾之忧
3. **云数据源支持**: 未来可能接入 BigQuery/Redshift，Redash 原生支持更好
4. **API 优先**: 适合自动化报表和集成开发
5. **性能**: 查询缓存机制，多用户并发体验更好

**风险点**:
- 无行级权限（需在查询层做过滤，增加复杂度）
- 可视化相对 Metabase 略少（但够用）

#### 🥈 备选: **Metabase**

**启用条件**:
- 团队内有非技术成员需要自助分析
- 嵌入式分析需求强烈（需将图表嵌入内部系统）
- 数据行级权限是硬需求

---

## 📦 部署建议（Redash）

### 快速启动（Docker）

```bash
# 1. 克隆官方 docker-compose
git clone https://github.com/getredash/redash.git
cd redash
cp .env.example .env

# 2. 修改配置（可选）
# 编辑 .env，设置数据库密码、域名等

# 3. 启动
docker-compose up -d

# 4. 初始化
docker-compose exec server bin/run
# 按提示创建管理员账号
```

### 生产就绪

- ✅ PostgreSQL 持久化（不要用内置 SQLite）
- ✅ Redis 持久化（缓存与会话）
- ✅ Nginx 反向代理 + SSL
- ✅ 定期备份（数据库 + 查询历史）
- ✅ 配置 SMTP（邮件通知）

预估资源:
- 内存: 2-4 GB
- CPU: 2 核
- 存储: 50 GB (根据数据量)

---

## 🔗 参考来源

- [Metabase GitHub](https://github.com/metabase/metabase)
- [Redash GitHub](https://github.com/getredash/redash)
- [Redash 开源部署指南](https://redash.io/help/open-source/setup/)
- [Metabase vs Redash 对比](https://www.google.com/search?q=metabase+vs+redash)

---

**状态**: ✅ Phase B 完成  
**下一阶段**: Phase C - Loki vs ELK (日志聚合)
