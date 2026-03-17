# Loki vs ELK - 日志聚合解决方案对比

**调研日期**: 2026-03-17  
**调研人**: 小深 (xiaoshen)  
**领域**: 日志聚合与可观测性

---

## 📊 核心数据对比

| 维度 | Loki (Grafana) | ELK 堆栈 (Elastic) |
|------|----------------|-------------------|
| **GitHub Stars** | 27,819 ⭐ | Elasticsearch: 76,383<br>Logstash: 14,806<br>Kibana: 21,004 |
| **主要语言** | Go | Java (ES) + Java (Logstash) + TypeScript (Kibana) |
| **许可证** | AGPL-3.0 | Elastic License (SSPL) ❗ |
| **架构** | 轻量索引 + 云存储 | 全文倒排索引 + 本地存储 |
| **官网** | grafana.com/loki | elastic.co |

---

## 🎯 核心定位

### Loki
> "Like Prometheus, but for logs."

**关键词**: 轻量、低成本、Prometheus 生态、指标日志统一

### ELK
> Elastic Stack: Elasticsearch (存储) + Logstash (处理) + Kibana (可视化)

**关键词**: 功能完整、企业级、全文检索、复杂分析

---

## 🏗️ 架构对比

### Loki 架构
```
日志源 → Promtail (收集) → Loki (索引) → Grafana (查询)
```

**特点**:
- **索引策略**: 只索引标签 (labels)，不索引日志内容
- **存储**: 对象存储 (S3/MinIO) +  boltshard/本地缓存
- **查询**: LogQL (类 PromQL 语法)
- **资源占用**: 低 (相比 ELK)

### ELK 架构
```
日志源 → Logstash/Filebeat → Elasticsearch → Kibana
```

**特点**:
- **索引策略**: 全文倒排索引（所有内容可检索）
- **存储**: 本地磁盘 (LSM tree)
- **查询**: Elasticsearch DSL / KQL (Kibana Query Language)
- **资源占用**: 高 (内存/CPU 密集)

**资源消耗对比**:
| 场景 | Loki | ELK |
|------|------|-----|
| 索引大小 | ~1-5% of raw logs | ~100% of raw logs |
| CPU | 轻量 | 高 |
| 内存 | 中等 | 高 (ES heap 建议 32GB) |
| 存储成本 | 低 (对象存储) | 高 (SSD 推荐) |

---

## 📈 功能对比

### 日志收集
| 功能 | Loki | ELK |
|------|------|-----|
| **收集器** | Promtail (专有) | Filebeat/Fluentd/Logstash |
| **多源支持** | ✅ 系统/应用/Docker/K8s | ✅ 极丰富（500+ 插件） |
| **解析能力** | 正则/JSON/日志格式 | Logstash 强大过滤/转换 |
| **多行日志合并** | ✅ | ✅ (Logstash) |

**Logstash 优势**: 数据处理管道强大，可做 enrich、过滤、格式转换

---

### 查询与搜索

#### Loki
- **查询语言**: LogQL (标签匹配 + 过滤器)
- **内容搜索**: 支持但不推荐（性能差）
- **范围查询**: 时间范围 + 标签选择
- **聚合**: 计数、分位数、TopN、Rate
- **实时性**: 高（秒级）

**示例**:
```logql
{app="nginx"} |= "error"
{job="mysql"} | json | status_code != 200
```

#### ELK
- **查询语言**: KQL / Elasticsearch DSL
- **全文检索**: ✅ Lucene 倒排索引
- **复杂查询**: 模糊匹配、短语、通配符、正则
- **聚合**: 极丰富（metrics/terms/histogram/...）
- **实时性**: 亚秒级

**示例**:
```
status_code:500 AND message:"timeout"
```

**搜索能力**: ELK >> Loki (全文搜索场景)

---

### 可视化

| 功能 | Loki | ELK |
|------|------|-----|
| **可视化工具** | Grafana (需搭配) | Kibana (原生) |
| **仪表板** | ✅ 强大（Grafana 生态） | ✅ 完整 |
| **图表类型** | Grafana 支持所有 | Kibana 丰富（Lens/Visualize/Dashboard） |
| **告警** | Grafana Alerting | Kibana Alerting + ElastAlert (社区) |
| **机器学习** | ❌ | ✅ (Elastic ML 功能) |

**可视化**: 平手（Grafana vs Kibana 各有特色）

---

### 权限与安全

| 功能 | Loki | ELK |
|------|------|-----|
| **RBAC** | Grafana 管理 | Kibana Spaces + 角色 |
| **多租户** | ✅ (通过 Grafana Org) | ✅ (Security plugin) |
| **数据加密** | 传输加密 | 传输 + 磁盘加密 (Elasticsearch) |
| **审计日志** | Grafana 审计 | ✅ Elasticsearch audit log |
| **SSO** | Grafana 支持 | ✅ (Elastic 企业版更强) |

**企业安全**: ELK > Loki (Elastic 商业特性)

---

## 📜 许可证分析

### Loki
- **许可证**: AGPL-3.0
- **限制**:
  - 网络服务需开源修改（传染性）
  - 商业使用允许
- **托管版本**: Grafana Cloud Loki（Grafana 提供）

### ELK
- **许可证**: Elastic License (SSPL) ❗
- **限制**:
  - **禁止**提供 ELK 作为托管服务（竞争）
  - 商业使用需购买商业许可
  - 功能分界：基本功能免费，高级功能（Security/Machine Learning）需订阅
- **托管版本**: Elastic Cloud（官方托管）

**许可证风险评级**:
- **Loki**: ⚠️ 中等（AGPL 注意网络服务）
- **ELK**: 🔴 高（SSPL + Elastic License 限制多）

---

## 💰 成本考量

### 直接成本
| 项目 | Loki | ELK |
|------|------|-----|
| **软件许可** | 免费 (AGPL) | 基础免费，高级功能付费 |
| **基础设施** | 低（存储成本低） | 高（SSD + 大内存） |
| **运维复杂度** | 低（组件少） | 高（三组件调试） |

**运维成本**: ELK 明显更高（资源 + 人力）

---

### 隐性成本
| 成本项 | Loki | ELK |
|--------|------|-----|
| **学习曲线** | 低（LogQL 类似 PromQL） | 中高（KQL + ES 概念） |
| **故障排查** | 简单（组件少） | 复杂（集群分片、GC 问题） |
| **扩展性** | 水平扩展简单 | 需要分片规划 |

---

## 🎪 典型使用场景

### Loki 适合
- ✅ 微服务日志聚合（K8s 环境）
- ✅ 已用 Prometheus + Grafana，希望统一查询
- ✅ 资源受限（中小企业）
- ✅ 快速部署，不想维护复杂系统
- ✅ 主要是基于标签的日志筛选（而非全文搜索）

### ELK 适合
- ✅ 全文搜索是刚需（如错误堆栈、用户输入）
- ✅ 复杂日志分析（多级聚合、异常检测）
- ✅ 安全合规（审计日志、完整历史）
- ✅ 已有 Elastic 投资或团队熟悉 ES
- ✅ 需要商业支持（Elastic 订阅）

---

## 📦 部署建议

### Loki (推荐配置)
```yaml
# docker-compose.yml (简化)
services:
  loki:
    image: grafana/loki:latest
    config:
      schema: v11
      storage:
        boltdb_shipper:
          active_index_directory: /loki/boltdb-shipper-active
          cache_location: /loki/boltdb-shipper-cache
          shared_store: s3
        aws:
          s3: s3://loki-bucket
          region: us-east-1
    resources:
      memory: 2Gi
      
  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - /etc/machine-id:/etc/machine-id

  grafana:
    image: grafana/grafana:latest
    # 配置 Loki 数据源
```

**资源预估**:
- Loki: 2-4 GB RAM
- Promtail: 512 MB per instance
- 存储: 对象存储（S3/MinIO）按日志量计费

---

### ELK (生产参考)
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false  # 生产需开启
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 16g
    volumes:
      - es-data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.12.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    ports:
      - "5601:5601"
```

**资源预估**:
- Elasticsearch: 32 GB heap + SSD 存储
- Logstash: 4 GB RAM per pipeline
- Kibana: 2 GB RAM

**⚠️ 注意**: ELK 资源需求远高于 Loki，小团队运维压力大。

---

## 🔗 参考来源

- [Loki GitHub](https://github.com/grafana/loki)
- [Elasticsearch GitHub](https://github.com/elastic/elasticsearch)
- [Logstash GitHub](https://github.com/elastic/logstash)
- [Kibana GitHub](https://github.com/elastic/kibana)
- [Loki vs ELK 对比](https://grafana.com/blog/2021/03/25/loki-vs-elasticsearch/)

---

## ✅ 对比总结

| 场景 | 推荐 | 理由 |
|------|------|------|
| **资源有限** | Loki | 存储成本低 90%+，内存占用少 |
| **全文搜索** | ELK | Lucene 倒排索引无可替代 |
| **K8s 环境 + Prometheus 生态** | Loki | 标签体系统一，Grafana 集成 |
| **复杂日志处理管道** | ELK | Logstash 插件生态强大 |
| **许可证合规** | 两者皆需注意<br>Loki: AGPL<br>ELK: SSPL + Elastic | 评估法律风险 |
| **快速上线** | Loki | 组件少，配置简单 |
| **已有 Elastic 团队经验** | ELK | 复用技能，降低学习成本 |

---

**状态**: Phase C 调研完成  
**下一步**: Phase D - Bull vs Agenda (任务调度)
