# 23 · 在线 DDL 与大表变更（Online DDL & Large Table Change）

> 千万/亿级大表加字段、加索引，**直接 `ALTER` 可能锁表、拖垮主从、撑爆磁盘**。要懂 MySQL 8.0 Online DDL 的三种 `ALGORITHM`（INSTANT/INPLACE/COPY）与 `LOCK` 级别，以及为什么生产还要用 **gh-ost / pt-osc** 这类外部工具。面试重要度 ⭐⭐ 常考（架构/DBA 岗高频）。

## 📖 核心原理

### 一、"千万级大表直接 ALTER 会怎样"——先看它到底做了什么

一条 `ALTER TABLE` 的代价，取决于它走 MySQL 内部三种算法中的哪一种：

| ALGORITHM | 是否重建表 | 是否允许并发 DML | 典型操作 | 耗时 |
|---|---|---|---|---|
| **INSTANT**（8.0.12+） | 否，只改**数据字典元数据** | 允许（几乎不阻塞） | 加列（末尾）、改列默认值、重命名列、加/删虚拟列 | **毫秒级** |
| **INPLACE** | 在 InnoDB 内部原地重建（不生成 Server 层临时表），但可能**rebuild 聚簇索引** | 大多允许 DML | 加二级索引、改列可空性、加/删主键 | 分钟~小时（随表大小） |
| **COPY** | 是，**复制整张表**到新表再 rename | **不允许**（全程锁表，只读甚至只加 MDL） | 改列类型、改字符集、部分主键操作 | 很慢，最危险 |

**直接 ALTER 的四大风险（面试要能一口气说全）：**

1. **锁表 / 阻塞 DML**：走 COPY 算法时全程 `LOCK=SHARED` 甚至更强，期间该表**写被阻塞**（甚至读也受影响），千万行拷贝几十分钟，业务大面积超时。
2. **MDL（元数据锁）等待**：即便走 INSTANT/INPLACE，`ALTER` 开始要拿表的 **MDL 写锁**。如果表上有**未提交的长事务/长查询**持有 MDL 读锁，`ALTER` 会被卡住，而它又会**阻塞后续所有对该表的新查询**——瞬间连接堆积、线程池打满，这是比"慢"更致命的雪崩。
3. **主从延迟**：DDL 在主库执行完才写 binlog 传到从库**串行重放**，从库执行同样耗时几十分钟，期间**主从延迟飙升**，读写分离的从库读到大面积旧数据。
4. **磁盘与 undo/redo 压力**：INPLACE rebuild / COPY 要额外一份表空间（原表大小的 1~2 倍），大表可能**撑爆磁盘**；长事务还会让 undo 膨胀、redo 频繁刷盘。

> **一句话结论**：在 **MySQL 8.0，末尾加一个可空列 / 带默认值的列**通常走 **INSTANT，毫秒完成**，可以直接 alter；但**加索引、改列类型、5.7 及更早版本加列**会 rebuild 或 copy，千万级表**绝不能业务高峰直接 alter**，要用 Online DDL 显式指定算法 + 低峰执行，或上 gh-ost/pt-osc。

### 二、MySQL 8.0 Online DDL：显式指定 ALGORITHM 与 LOCK

```sql
-- 8.0 末尾加列：INSTANT，秒级，几乎不阻塞（推荐这样显式声明，失败即报错而非偷偷降级）
ALTER TABLE orders ADD COLUMN remark VARCHAR(255) NULL, ALGORITHM=INSTANT;

-- 加二级索引：INPLACE + LOCK=NONE，允许并发 DML，但仍会扫描/构建索引
ALTER TABLE orders ADD INDEX idx_user (user_id), ALGORITHM=INPLACE, LOCK=NONE;

-- 改列类型：只能 COPY，会锁表——大表禁止直接这么干
ALTER TABLE orders MODIFY COLUMN amount BIGINT, ALGORITHM=COPY;
```

**关键技巧**：显式写 `ALGORITHM=INSTANT`（或 `INPLACE, LOCK=NONE`），如果该操作**不支持**这个算法，MySQL 会**直接报错**，而不是悄悄降级成 COPY 把库锁死——这是生产保命写法。

**INSTANT 的限制（易被追问）**：① 只能加在**行末尾**（8.0.29 前）；8.0.29+ 支持任意位置加列但仍是即时；② 有次数/场景限制（一张表 INSTANT 加列累计有上限，超了需一次 rebuild 才能继续）；③ 不能同时做 INSTANT 与非 INSTANT 操作。

### 三、为什么 8.0 有了 Online DDL，生产还要 gh-ost / pt-osc？

因为原生 Online DDL 仍有绕不过的坑：

- **仍会 rebuild 的操作**（改类型、部分加索引场景）在原生 DDL 下依旧长时间占资源、**主从串行重放导致从库长延迟**——原生 DDL 对"主从延迟"这件事**完全不可控**。
- **MDL 抢锁**：原生 DDL 拿 MDL 是"要么等、要么把后面全堵住"，无法优雅避让长事务。
- 外部工具能做到**可暂停、可限流、可随时中止、延迟自动降速**，对线上更友好，是大厂 DBA 的标准做法。

## 🔄 原理图 / 流程剖析

**pt-online-schema-change（触发器方案）与 gh-ost（binlog 方案）的核心都是"影子表 + 分块拷贝 + 原子 rename"，区别在如何捕获增量：**

```mermaid
flowchart TD
    subgraph PT["pt-osc：触发器捕获增量"]
      A1[1. 建空影子表 _new] --> A2[2. 在原表建 INS/UPD/DEL 触发器]
      A2 --> A3[3. 分 chunk 拷贝存量行到 _new]
      A3 --> A4[4. 触发器实时把增量同步到 _new]
      A4 --> A5[5. 原子 RENAME 原表↔_new]
      A5 --> A6[6. 删旧表 + 删触发器]
    end
    subgraph GH["gh-ost：解析 binlog 捕获增量"]
      B1[1. 建空影子表 _gho] --> B2[2. 伪装成从库拉原表 binlog]
      B2 --> B3[3. 分 chunk 拷贝存量行]
      B3 --> B4[4. 解析 binlog 把增量 apply 到 _gho]
      B4 --> B5[5. 低峰原子 cut-over 切换]
    end
```

**关键差异对比：**

| 维度 | pt-online-schema-change | gh-ost（GitHub 出品） | 原生 Online DDL |
|---|---|---|---|
| 增量捕获 | **触发器**（写在原表上） | **解析 binlog**（无触发器） | 引擎内部 |
| 对主库额外负担 | 触发器使每次写放大（同步写影子表），高并发下压力大 | **无触发器**，几乎不干扰原表写 | 视算法而定 |
| 可控性 | 有限流、可中止 | **可暂停/限流/交互式改参数**，能感知主从延迟自动降速 | 基本不可控 |
| 外键 | 支持较弱（外键处理麻烦） | 对外键支持也有限 | 支持 |
| cut-over 锁 | RENAME 瞬间元数据锁 | cut-over 瞬间锁，做得更精细 | MDL |
| 依赖 | 触发器权限 | 需要能读 binlog（row 格式） | 无 |

> gh-ost 的最大卖点：**无触发器**（触发器会让原表每次 DML 变慢、且和已有触发器冲突），改为像一个从库一样订阅 binlog 来同步增量，对主库侵入小、更可控，因此近年更受青睐。

**典型 gh-ost 命令：**

```bash
gh-ost \
  --host=127.0.0.1 --database=shop --table=orders \
  --alter="ADD COLUMN remark VARCHAR(255) NULL" \
  --max-load=Threads_running=50 \       # 负载过高自动降速
  --max-lag-millis=1500 \               # 从库延迟超 1.5s 自动暂停拷贝
  --chunk-size=1000 \                    # 每批拷贝行数
  --initially-drop-ghost-table \
  --allow-on-master --execute
```

## 🔑 面试要点

- **千万级表直接 alter 的风险四件套**：①COPY 算法锁表阻塞 DML ②MDL 被长事务卡住并连锁堵住后续查询 ③主从串行重放导致从库大延迟 ④额外表空间撑爆磁盘。
- **MySQL 8.0 三种算法**：INSTANT（改元数据、毫秒、末尾加列）＞INPLACE（原地/可能 rebuild、多数允许 DML）＞COPY（拷贝整表、锁表、最慢）。
- **生产保命写法**：显式 `ALGORITHM=INSTANT`（或 `INPLACE, LOCK=NONE`），不支持就报错，避免悄悄降级成 COPY 锁库。
- **8.0 加列 vs 5.7 加列**：8.0 末尾加列 INSTANT 秒级；5.7 及更早即使"Online" 也要 rebuild，千万级会很慢——版本差异是高频追问点。
- **为什么还要 gh-ost/pt-osc**：原生 DDL 对**主从延迟不可控、无法限流/暂停**；工具用影子表 + 分块拷贝 + 增量同步 + 原子切换，可感知负载/延迟自动降速。
- **pt-osc vs gh-ost 核心区别**：前者用**触发器**捕获增量（写放大、侵入原表），后者**解析 binlog**（无触发器、侵入小、可控性强）。

## ❓ 高频面试题

**Q：给一张千万级的大表加一个字段，直接 `ALTER TABLE ... ADD COLUMN` 会发生什么？**
A：分版本和操作看。MySQL **8.0 在末尾加一个可空/带默认值的列**会走 **INSTANT 算法**，只改数据字典元数据，**毫秒级完成、几乎不阻塞**，可以直接加。但如果是 **5.7 及更早**，或者加列涉及 rebuild（如加在中间、某些默认值场景），会走 INPLACE rebuild 甚至 COPY：**拷贝/重建整张表几十分钟**，期间视算法阻塞 DML，还会导致**主从延迟飙升**（从库串行重放同样慢）、**占用双倍表空间**。更隐蔽的杀手是 **MDL**：ALTER 要拿表的元数据写锁，若此刻有未提交长事务持有该表 MDL，ALTER 被卡住并**连带阻塞后续所有对该表的新请求**，瞬间连接雪崩。所以生产做法：8.0 显式 `ALGORITHM=INSTANT` 直接加；不支持 INSTANT 的变更用 **gh-ost/pt-osc** 在低峰执行，并先确认无长事务。

**Q：MySQL Online DDL 的 INPLACE 和 COPY 有什么区别？INSTANT 又是什么？**
A：**COPY** 会在 Server 层建一张临时表，把数据一行行拷过去再 rename，全程基本锁表、不允许并发写，最慢最危险。**INPLACE** 在 InnoDB 存储引擎内部原地操作（不经 Server 层临时表），多数支持并发 DML（`LOCK=NONE`），但像"加二级索引/改主键"仍需重建聚簇索引，大表依然耗时。**INSTANT**（8.0.12+）最轻，只修改表的**数据字典元数据**、不碰数据行，毫秒完成，典型如末尾加列、改默认值、重命名列。优先级：能 INSTANT 就 INSTANT，其次 INPLACE+LOCK=NONE，尽量避免 COPY。

**Q：既然 8.0 有 Online DDL 了，为什么大厂还用 gh-ost / pt-osc？它俩又有什么区别？**
A：原生 Online DDL 有两个绕不过的问题：一是**对主从延迟完全不可控**——需要 rebuild 的 DDL 在从库串行重放会造成长时间延迟；二是**无法限流、暂停、中止**，MDL 抢锁也可能连锁阻塞。gh-ost/pt-osc 用"影子表 + 分块拷贝存量 + 同步增量 + 原子 rename 切换"的思路，能**感知负载和主从延迟自动降速、可暂停可回退**，对线上更安全。两者区别在**增量捕获方式**：pt-osc 在原表上建**触发器**同步增量（对原表写有放大、侵入性强、和已有触发器冲突）；gh-ost **伪装成从库解析 binlog** 来同步增量、**无触发器**，对主库侵入小、可控性更好，所以更受欢迎。

**Q：ALTER 大表时你会怎么规避 MDL 导致的雪崩？**
A：① 执行 DDL 前先查 `information_schema.innodb_trx` / `SHOW PROCESSLIST` 确认**没有长事务/长查询**持有该表 MDL；② 给 DDL 设**较短的 `lock_wait_timeout`**，抢不到锁快速失败重试，而不是无限等并堵住后面；③ 选**业务低峰**执行；④ 用 gh-ost 这类工具，把风险集中在**极短的 cut-over 瞬间**而非整个变更过程。

## ⚠️ 易错点 / 加分项

- ⚠️ **别以为"Online DDL"就等于"不锁表、随便加"**——它只是相对旧版减少阻塞，COPY 算法照样锁，INPLACE rebuild 照样拖主从。
- ⚠️ **INSTANT 加列有次数上限**：同一张表反复 INSTANT 加列到一定次数后，必须做一次 rebuild 才能继续，别在设计里依赖无限 INSTANT。
- ⚠️ **gh-ost/pt-osc 拷贝期间磁盘要留够空间**（多一份表大小），且 cut-over 瞬间仍有短暂锁，别在秒杀峰值切换。
- ✅ **加分**：能点出"MDL 被长事务阻塞并连锁堵住后续请求"这个**比慢更致命的雪崩机制**，比只答"锁表慢"高一个层次。
- ✅ **加分**：提到 8.0.29+ INSTANT 支持任意位置加列、以及 `lock_wait_timeout` + 低峰 + 无长事务的组合规避策略，体现实战经验。
- 🔗 关联：主从延迟见 [`19-master-slave-replication.md`](19-master-slave-replication.md)、[`20-read-write-split.md`](20-read-write-split.md)；MDL/锁体系见锁相关章节；分库分表后的 DDL 治理见 [`21-sharding.md`](21-sharding.md)。
