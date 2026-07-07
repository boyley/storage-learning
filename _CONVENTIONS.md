# 存储学习合集 · 统一规范（所有 sub-agent 必须遵守）

> 本文件是本工程唯一风格标准。任何 agent 生成内容前先读本文件。目标：**资深级别、面试导向、讲透原理与底层机制、一个知识点一个 md、按顺序编号**。

## 一、目录结构（一个存储产品一个文件夹 + 扁平知识点）

```
storage-learning/
├── README.md                 ← 总览 + 知识地图 + 面试冲刺路线
├── _CONVENTIONS.md           ← 本文件
├── mysql/                    ← 一个存储产品一个文件夹（可扩展：mysql / redis / 等）
│   ├── README.md             ← 该产品的知识点索引 + 学习路线
│   ├── 01-architecture.md    ← 一个知识点一个 md（扁平编号）
│   └── ...
└── redis/
    ├── README.md
    ├── 01-overview.md
    └── ...
```

将来可加 `elasticsearch/`、`mongodb/`、`kafka/` 等同级产品目录。

## 二、命名

- 产品目录：小写英文，如 `mysql`、`redis`。
- 知识点文件：`NN-knowledge-point.md`（两位数字 + kebab-case 英文），如 `12-mvcc.md`。
- 编号 = 推荐阅读/复习顺序，由架构/原理到进阶/实战。

## 三、每个知识点 md 固定结构（资深面试导向）

```markdown
# NN · 知识点中文名（English Name）

> 一句话核心 + 面试重要度（⭐⭐⭐ 高频 / ⭐⭐ 常考 / ⭐ 了解）。

## 📖 核心原理
（讲清概念本质**与底层机制**；资深级别要讲到数据结构/执行流程/参数，不只停留在"是什么"。2~6 段。）

## 🔄 原理图 / 流程剖析
（关键机制用 Mermaid 图或对比表讲清：B+树/MVCC 版本链/锁范围/主从复制/持久化流程/集群分槽等。复杂主题必配图。）

## 🔑 面试要点
（bullet 列出必背关键点，含"面试怎么答"，5~8 条。）

## ❓ 高频面试题
**Q：……？**
A：……（2~4 个资深级问题，问题加粗，答案到位、点出深度。）

## ⚠️ 易错点 / 加分项
（3~5 条：常见误区、坑、答出来能拉开差距的深入点。）
```

> 资深级别的核心区别：**必须讲 how/why 与底层机制**（B+树回表、Buffer Pool、MVCC ReadView、Next-Key Lock、redo/undo/binlog 两阶段提交、RDB fork COW、缓存一致性、Cluster 分槽），而不是命令/配置罗列。

## 四、版本基准

- MySQL：**8.0**，存储引擎以 **InnoDB** 为主线；涉及版本差异（如 5.7 vs 8.0、隔离级别默认 RR）要点明。
- Redis：**7.x**；涉及版本差异（如 ziplist→listpack、Redis 6 多线程 IO、混合持久化 4.0+）要点明。

## 五、内容语言与质量底线

- 讲解一律**中文**；命令、参数、数据结构名用英文（如 `Buffer Pool`、`Next-Key Lock`、`SETNX`、`maxmemory-policy`）。
- 对照官方文档与权威资料整理，参数给真实名（如 `innodb_flush_log_at_trx_commit`、`appendfsync`），示例 SQL/命令可运行。
- 与 Java/JVM 相关处（如 Redisson、连接池、分布式锁在业务侧）可一句话引用 `../../java-learning`、`../../spring-learning`，不重复展开。
- 每个 md 聚焦**一个**知识点，篇幅充实但不灌水；突出"怎么答 + 加分点 + 踩坑"。

## 六、覆盖主线

- **MySQL**：架构与执行流程 → 存储引擎/InnoDB → 索引(B+树/回表/优化) → EXPLAIN/SQL 优化 → 事务/隔离/MVCC → 锁/死锁 → redo/undo/binlog/两阶段提交 → 主从复制/读写分离 → 分库分表/高可用/在线 DDL。
- **Redis**：为什么快/单线程/IO 多路复用 → 数据类型与底层结构 → 持久化(RDB/AOF/混合) → 过期与淘汰 → 缓存三大问题与一致性 → 分布式锁 → 主从/哨兵/Cluster → 事务/pipeline/大key热key/实战。
