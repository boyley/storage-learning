# 22 · 实战与最佳实践（Best Practices）

> 把前面所有知识点收敛成一份可落地的工程清单：键怎么设计、命令怎么用、内存/持久化/高可用怎么配、线上怎么排查、安全怎么兜底。面试重要度：⭐⭐ 常考（用来考查你有没有真的踩过生产的坑）。

## 📖 核心原理

这一篇没有新的底层机制，价值在于**把散点知识组织成一套「默认就该这么做」的工程规约**。面试里被问「你们线上 Redis 怎么用的 / 踩过什么坑」，本质是在验证前面几篇（数据结构、持久化、淘汰、缓存问题、高可用）你是不是真理解、能不能落到规范上。下面按六个维度系统汇总，每条都尽量给出**为什么**（背后的机制在哪一篇），而不是空喊口号。

一个统领性的心智模型：**Redis 是内存数据库，一切最佳实践都围绕「省内存、降延迟、保命（不丢数据/不宕全站）」三件事展开**。键设计和数据结构选型是省内存，命令选择和 pipeline 是降延迟，持久化+高可用+安全是保命。带着这三条主线去记，比背清单更牢。

## 🔄 分类速查表 / 规约剖析

### 1）键设计（Key Design）——省内存的第一道关

| 规约 | 推荐做法 | 为什么 / 底层原因 |
|---|---|---|
| 命名规范 | `业务:模块:id`，冒号分隔，如 `order:detail:1001`、`user:session:abc` | 冒号是社区约定的层级分隔符（`RedisInsight`/`RDM` 会按 `:` 折叠成树），可读、可按前缀 `SCAN` 批量运维 |
| 控制 key 长度 | 语义清晰前提下尽量短，别把整句话当 key | 每个 key 都是一个 `sds` + `dictEntry`，key 本身也占内存；千万级 key 时几字节差异会放大成几十 MB |
| 避免大 key | 单个 String ≤ 10KB，集合类元素数 ≤ 5000（经验值），拆分或用 Hash 分片 | 大 key 删除/迁移/序列化会阻塞主线程，是慢查询和 Cluster 迁移卡顿的头号原因，详见 [20-bigkey-hotkey](20-bigkey-hotkey.md) |
| 选对数据结构与编码 | 小对象让它落到紧凑编码：Hash/List/ZSet 元素少时用 `listpack`，Set 全整数用 `intset` | 紧凑编码用连续内存、省指针开销；一旦超过 `hash-max-listpack-entries`/`-value` 等阈值会转成 `hashtable`/`skiplist`，内存翻几倍，详见 [03-data-structures-internal](03-data-structures-internal.md) |
| 用 Hash 聚合而非海量顶层 key | 100 万条字段用一个（或分片的）Hash 存，而非 100 万个 String | 顶层 key 多一份全局 `dict` 开销；聚合进 Hash 且保持在 `listpack` 编码内，内存能省几倍（前提是别聚成大 key） |

### 2）命令使用（Command Usage）——别让一条命令打爆主线程

| 场景 | 禁用/慎用 | 替代方案 | 原因 |
|---|---|---|---|
| 遍历键空间 | `KEYS *`（O(n) 全库扫描、阻塞） | `SCAN`（游标分批、O(1) 每次） | `KEYS` 单线程下会卡住所有请求；`SCAN` 用 reverse binary iteration 保证 rehash 期间不漏不重（弱一致） |
| 清库 | `FLUSHALL`/`FLUSHDB`（同步阻塞） | `FLUSHALL ASYNC`（后台释放） | 同步版本要在主线程逐个释放对象，库大就长时间阻塞 |
| 取集合全量 | 大范围 `SMEMBERS`/`HGETALL`/`LRANGE 0 -1` | `SSCAN`/`HSCAN` 或分页 | 一次返回百万元素既阻塞主线程又打爆网络和客户端内存 |
| 批量读写 | N 次单命令来回（N 次 RTT） | `MGET`/`MSET` 或 pipeline 合并 | 减少网络往返，吞吐量数量级提升，详见 [15-pipeline](15-pipeline.md) |
| 删除大 key | `DEL bigkey`（同步释放，阻塞） | `UNLINK bigkey`（后台线程回收内存） | `DEL` 在主线程释放百万元素会卡几百 ms；`UNLINK` 只解链、内存回收交给 `lazyfree` 后台线程 |
| 全局慢命令 | `SORT`（无索引）、`ZRANGEBYSCORE` 大区间 | 预排序落库 / 限制返回条数 | 复杂度 O(n·logn) 级，大集合上是隐形炸弹 |

> 记忆口诀：**「遍历用 SCAN，批量用 pipeline，删除用 UNLINK，清库加 ASYNC」**。凡是复杂度带 n 且 n 不可控的命令，上线前都要问一句「n 最大能到多少」。

### 3）过期与内存（Expiration & Memory）

| 规约 | 做法 | 原因 / 链接 |
|---|---|---|
| 都设合理 TTL | 缓存类 key 一律带过期，别当持久存储裸放 | 无 TTL 的 key 只能靠内存淘汰兜底，容易堆积；配合惰性+定期删除机制 |
| 配 `maxmemory` + 淘汰策略 | 设物理内存的 70%~80%，纯缓存用 `allkeys-lru`/`allkeys-lfu` | 留冗余给 fork COW 和复制缓冲区；`volatile-*` 若没带 TTL 的 key 会退化成 `noeviction` 写报错，详见 [10-eviction](10-eviction.md) |
| 防缓存雪崩 | TTL 加随机扰动（如 `基础 + rand(0, 300s)`） | 大批 key 同一时刻过期会瞬间穿透到 DB，随机化把过期时间摊开，详见 [11-cache-problems](11-cache-problems.md) |
| 大 key 过期风险 | 大集合 key 过期时删除会阻塞 | 开 `lazyfree-lazy-expire yes` 让过期删除也走后台线程 |

### 4）持久化与高可用（Persistence & HA）

| 维度 | 推荐配置 | 原因 / 链接 |
|---|---|---|
| 持久化方式 | RDB + AOF **混合持久化**（`aof-use-rdb-preamble yes`，4.0+ 默认开） | AOF 重写时前半段写 RDB 二进制、后半段追 AOF 增量：既有 RDB 的快加载，又有 AOF 的秒级不丢，详见 [08-persistence-hybrid](08-persistence-hybrid.md) |
| AOF 刷盘 | `appendfsync everysec`（默认） | `always` 太慢、`no` 靠 OS 最多丢 30s；`everysec` 是性能与安全的平衡点，最多丢 1s |
| 主从复制 | 至少 1 主 1~2 从，`replica-read-only yes` | 读扩展 + 故障时有热备可切，复制原理详见 [17-replication](17-replication.md) |
| 自动故障转移 | 中小规模用 **哨兵（Sentinel）**，3 个哨兵起步 | 哨兵负责监控+选主+通知客户端，奇数个防脑裂，详见 [18-sentinel](18-sentinel.md) |
| 水平扩展 | 数据量/写入超单机上限用 **Cluster** | 16384 槽分片、去中心化 gossip，详见 [19-cluster](19-cluster.md) |
| 防脑裂 | `min-replicas-to-write 1` + `min-replicas-max-lag 10` | 主库确认至少 1 个从库在线才接受写，减少脑裂期间的数据丢失 |

### 5）性能与排查（Performance & Troubleshooting）

| 工具 / 指标 | 用途 | 关注点 |
|---|---|---|
| `slowlog get` / `slowlog-log-slower-than` | 慢查询日志（默认 >10000 微秒记录） | 抓 O(n) 命令、大 key 操作；定期看有没有 `KEYS`/大 `HGETALL` |
| `INFO` 分节 | 全局健康体检 | `used_memory`/`mem_fragmentation_ratio`（内存碎片，>1.5 需关注）、`keyspace_hits`/`misses`（命中率）、`connected_clients`、`instantaneous_ops_per_sec`（QPS） |
| `LATENCY DOCTOR` / `LATENCY HISTORY` | 延迟尖刺诊断 | fork、AOF 刷盘、大 key、swap 引起的毛刺定位 |
| `--bigkeys` / `--hotkeys` / `MEMORY USAGE key` | 采样大 key、热 key | `--hotkeys` 需先开 LFU；根治见 [20-bigkey-hotkey](20-bigkey-hotkey.md) |
| 客户端连接池 | 复用连接、避免频繁建连 | Jedis/Lettuce 配 `maxTotal`/`maxIdle`，别每次请求新建连接（TCP 握手+auth 开销大） |
| `redis-benchmark` | 压测基线 | 上线前测清楚单实例 QPS 天花板 |

> 排查心法：**慢 → 先看 `slowlog` 和 `LATENCY`；内存涨 → 看 `INFO memory` + `--bigkeys`；命中率低 → 看 `keyspace_hits/misses` 比值；抖动 → 排查 fork/AOF 刷盘/swap**。

### 6）安全（Security）——别公网裸奔

| 措施 | 做法 | 说明 |
|---|---|---|
| 认证 | `requirepass` 设强密码，6.0+ 用 **ACL** 分角色 | ACL 可按用户限制可执行命令和可访问 key 前缀，最小权限 |
| 禁危险命令 | `rename-command FLUSHALL ""`（禁用）或改名 | 把 `KEYS`/`FLUSHALL`/`CONFIG`/`SHUTDOWN` 改成随机名，防误操作和攻击 |
| 网络隔离 | `bind` 内网 IP + `protected-mode yes` | 绝不 `bind 0.0.0.0` 裸奔公网；配合安全组/防火墙只放业务网段 |
| 端口 | 改默认 `6379`，别用众所周知端口 | 降低被扫描爆破概率 |

> 血泪教训：公网裸奔 + 无密码的 Redis 是勒索/挖矿病毒的经典入口（`CONFIG SET dir` + `SAVE` 可往任意路径写文件植入后门）。**内网 + 密码/ACL + 禁危险命令**是底线三件套。

## 🔑 面试要点

- **键设计三原则**：`业务:模块:id` 冒号分层命名、控制长度、避免大 key；小对象靠阈值参数（`hash-max-listpack-entries` 等）保持紧凑编码省内存。
- **命令四口诀**：遍历用 `SCAN` 不用 `KEYS`、批量用 `pipeline`/`MGET`、删除大 key 用 `UNLINK`、清库加 `ASYNC`——核心都是**别让单条命令长时间占住单线程**。
- **内存四件套**：全设 TTL + `maxmemory`（留 20%~30% 冗余）+ 合适淘汰策略 + 随机 TTL 防雪崩。
- **高可用组合**：混合持久化保数据、`appendfsync everysec` 保刷盘、主从+哨兵/Cluster 保可用、`min-replicas-*` 防脑裂。
- **排查工具链**：`slowlog`（慢命令）、`INFO`（内存/命中率/QPS/碎片率）、`LATENCY`（毛刺）、`--bigkeys/--hotkeys`（大热 key）、连接池（连接复用）。
- **安全底线**：`requirepass`/ACL + `rename-command` 禁危险命令 + `bind` 内网 + `protected-mode`，绝不公网无密码裸奔。
- **怎么答加分**：每条规约都能说出**背后的机制**（为什么 `KEYS` 阻塞、为什么大 key 危险、为什么 `volatile-*` 会退化），而不是背命令清单——这是资深和初级的分水岭。

## ❓ 高频面试题

**Q：你们线上 Redis 有哪些使用规范 / 踩过哪些坑？**
A：分层说更显体系。**键层面**：统一 `业务:模块:id` 命名、控制大 key（曾因一个几百万元素的 Hash 做 `HGETALL` 打满主线程，拆成分片 Hash + `HSCAN` 解决）；**命令层面**：禁 `KEYS`（用 `SCAN`）、大 key 删除全改 `UNLINK`、批量走 pipeline；**内存层面**：全设 TTL 且 TTL 加随机值防雪崩、`maxmemory` 只给物理内存 75% 留 fork 冗余、策略用 `allkeys-lru`；**高可用**：混合持久化 + 哨兵三节点 + `min-replicas-to-write` 防脑裂。踩过最典型的坑是缓存雪崩（一批 key 同秒过期打穿 DB）和大 key 阻塞，都在规范里补上了。

**Q：为什么线上要禁用 `KEYS` 命令？用什么替代？**
A：Redis 命令在**单线程**里串行执行，`KEYS pattern` 是 O(n) 全库扫描，key 多时会把主线程占住几百 ms 到秒级，期间所有其他请求全部排队甚至超时，直接引发雪崩。替代是 `SCAN`：它基于游标分批返回，每次只扫一小部分，用 reverse binary iteration 算法保证即使在 rehash 期间也不漏、不重（但可能重复，弱一致，业务要幂等处理）。同理 `HSCAN`/`SSCAN`/`ZSCAN` 替代大范围的 `HGETALL`/`SMEMBERS`。

**Q：`DEL` 和 `UNLINK` 有什么区别？删大 key 该用哪个？**
A：`DEL` 在**主线程同步**释放对象内存，删一个百万元素的集合要逐个 free，会阻塞几百 ms；`UNLINK` 只是把 key 从键空间**解链**（O(1) 从字典摘除），真正的内存回收交给 `lazyfree` 后台线程异步做，主线程几乎不阻塞。所以删大 key 一律用 `UNLINK`。相关的还有 `lazyfree-lazy-eviction`/`-expire`/`-server-del` 等参数，可以让淘汰、过期、`FLUSH` 的内存释放也都走后台线程。

**Q：`maxmemory` 为什么建议只设物理内存的 70%~80%？**
A：因为 RDB/AOF 重写要 `fork` 子进程，基于 COW（写时复制）共享内存页，父进程在重写期间持续写入会不断复制新页，最坏情况额外占用接近一份数据量的内存；此外主从复制的 replication backlog、客户端输出缓冲区也要占内存。如果 `maxmemory` 设成物理内存 100%，fork 期间就可能触发 OOM Killer 把 Redis 杀掉或大量 swap 导致延迟飙升。留 20%~30% 冗余是安全垫。

## ⚠️ 易错点 / 加分项

- **误区**：以为设了 `maxmemory` + 淘汰策略就万事大吉。若用 `volatile-*` 而 key 又都没设 TTL，淘汰时找不到候选会**退化成 `noeviction`**，内存满后写命令直接 `OOM` 报错——纯缓存务必用 `allkeys-*`（见 [10-eviction](10-eviction.md)）。
- **踩坑**：pipeline 不是万能药。**一个 pipeline 塞太多命令**会让单次响应变超大、客户端和服务端缓冲区暴涨，且 pipeline 内命令**非原子**（中间不保证不被打断），需要原子性得用 Lua 或事务（见 [15-pipeline](15-pipeline.md)）。
- **踩坑**：`SCAN` 是弱一致的——遍历过程中新增的 key 可能扫不到、已有的可能重复返回，业务侧逻辑必须幂等，别拿它当强一致快照用。
- **加分点**：热 key 的根治不能只靠 Redis——可以**本地缓存（多级缓存）** 挡一层、或把热 key **拆分成多个副本**（`hotkey#1`~`hotkey#N`）分散到不同 slot 打散压力（见 [20-bigkey-hotkey](20-bigkey-hotkey.md)）。
- **加分点**：内存碎片率 `mem_fragmentation_ratio` 长期 >1.5 说明碎片严重，4.0+ 可开 `activedefrag yes` 做主动碎片整理；<1 则说明用了 swap，是危险信号要立刻排查。
- **加分点**：`CONFIG SET` 的很多参数可**热更新**（如 `maxmemory`、`maxmemory-policy`、`slowlog-log-slower-than`），线上调优不必重启；但记得同步改配置文件，否则重启后失效。

## ✅ 上线前踩坑检查清单（Checklist）

一段汇总最容易翻车的点，上线前逐条过：

- [ ] **键**：命名统一 `业务:模块:id`？有没有大 key（`--bigkeys` 扫过）？集合类会不会无限增长（有没有裁剪/TTL）？
- [ ] **编码**：小对象是否落在 `listpack`/`intset` 紧凑编码内？阈值参数是否按业务调过？
- [ ] **命令**：代码里有没有 `KEYS`/大 `HGETALL`/`SMEMBERS`/无 `LIMIT` 的 `SORT`？删除是否用 `UNLINK`？批量是否走了 pipeline/MGET？
- [ ] **过期**：缓存 key 是否都设了 TTL？TTL 是否加了随机扰动防雪崩？
- [ ] **内存**：`maxmemory` 设了吗（≤ 物理内存 80%）？`maxmemory-policy` 是纯缓存友好的 `allkeys-*` 吗？`lazyfree-*` 开了吗？
- [ ] **持久化**：混合持久化开了吗？`appendfsync` 是 `everysec` 吗？磁盘空间够 AOF/RDB 写吗？
- [ ] **高可用**：有主从吗？哨兵/Cluster 节点是奇数吗？`min-replicas-to-write` 防脑裂配了吗？
- [ ] **排查**：`slowlog` 阈值合理吗？有没有接入 `INFO` 指标监控告警（内存/命中率/连接数/QPS/碎片率）？客户端用连接池了吗？
- [ ] **安全**：设密码/ACL 了吗？危险命令 `rename-command` 了吗？`bind` 内网 + `protected-mode yes` 了吗？端口改了吗？**确认没有公网裸奔**。
