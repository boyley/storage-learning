# 《Redis 源码剖析与实战》面试复习总结

> 按课程顺序整理，每讲三段式：**一句话核心 / 关键知识点 / 面试高频问答**。适合面试前快速扫读，不必再从头看原文。作者：蒋德钧。源码课，这里已提炼成**面试导向**（重原理与结论，源码函数点到为止）。

---

## 一、面试高频考点速查表（按主题）

Redis 面试高度集中在下面这几块，先看表定位，再翻到对应讲精读。

| 主题 | 一句话记忆 | 对应讲 |
|---|---|---|
| **整体架构** | 单线程 + 事件驱动 + 全局哈希表(dict)；redisObject + 8 种编码 | 01 |
| **数据结构** ★ | SDS、dict(渐进式rehash)、ziplist/quicklist/listpack、skiplist、intset | 02~07 |
| **IO 模型** ★ | IO 多路复用(epoll) + Reactor 单线程；6.0 多线程只做网络 IO | 09~13 |
| **单线程为何快** ★ | 纯内存 + IO 多路复用 + 无锁无上下文切换 + 高效结构 | 12 |
| **过期与淘汰** ★ | 惰性+定期删除；8 种淘汰策略；近似 LRU / LFU | 15~17 |
| **持久化** ★ | RDB(快照,fork+COW)、AOF(写后日志,重写)、混合持久化 | 18~20 |
| **主从复制** | 全量(psync)+增量(repl_backlog)、offset、runid | 21 |
| **哨兵 Sentinel** | 监控+故障转移；主观/客观下线；Raft 选 Leader | 22~25 |
| **Cluster 集群** | 16384 槽 + CRC16 + Gossip；MOVED/ASK 重定向 | 26~28 |
| **分布式锁** | SET NX EX 原子性、Lua 防误删、Redlock | 14 |

---

## 二、必背核心结论（背这些应付大部分 Redis 八股）

1. **Redis 为什么快**：① 纯内存操作；② IO 多路复用(epoll) 高效处理海量连接；③（命令执行）单线程避免锁竞争和线程上下文切换；④ 高效的底层数据结构。
2. **Redis 真的是单线程吗**：**命令执行是单线程**，但不止一个线程——还有后台线程 **BIO**（异步关闭文件、AOF 刷盘、lazyfree 大 key 删除）；**6.0 引入多线程 IO**（`io-threads`）只负责网络读写和协议解析，**命令执行仍是单线程**。
3. **8 种数据结构编码**：String→int/embstr/raw；List/Hash/ZSet/Set 各有 ziplist(listpack)/hashtable/skiplist/intset 等编码，按元素数量和大小自动转换。
4. **SDS（简单动态字符串）优于 C 字符串**：O(1) 取长度、二进制安全、杜绝缓冲区溢出、空间预分配 + 惰性释放减少内存重分配。
5. **渐进式 rehash**：dict 扩容时用两个哈希表 ht[0]/ht[1]，`rehashidx` 记录进度，每次操作顺带迁移一部分，**避免一次性 rehash 阻塞主线程**。负载因子超阈值触发扩容。
6. **ZSet 用 skiplist + hashtable 双结构**：hashtable 支持 O(1) 按成员点查，skiplist 支持 O(logN) 范围查询/排名；不用红黑树是因为跳表实现简单、范围查询方便、并发/内存可控。
7. **过期删除 = 惰性删除 + 定期删除**：访问时才判断是否过期（惰性）+ 周期性随机抽查删除（定期）；配合内存淘汰兜底。
8. **8 种内存淘汰策略**：noeviction（默认，不淘汰返错）、allkeys/volatile × lru/lfu/random，加 volatile-ttl。生产常用 **allkeys-lru** 或 **allkeys-lfu**。
9. **近似 LRU**：不维护全局链表（省内存），用 redisObject 的 lru 字段存访问时间戳，淘汰时**随机采样 `maxmemory-samples` 个** key，淘汰其中最久未用的。
10. **LFU**：lru 字段拆成 16bit 访问时间 + 8bit 访问计数，计数**按概率对数递增**且**随时间衰减**，解决"历史高频但近期不用"的缓存污染。
11. **RDB**：内存快照，`bgsave` **fork 子进程** + **写时复制(COW)**，主进程照常服务；文件紧凑、恢复快，但可能丢最后一次快照后的数据。
12. **AOF**：写后日志（先执行命令再记日志），`appendfsync` 三档：always（最安全最慢）/ **everysec（默认，折中）** / no。AOF 越写越大靠 **bgrewriteaof 重写**（用当前数据生成最小命令集），重写期间新写命令进 **AOF 重写缓冲区**。**混合持久化**：RDB 全量 + AOF 增量。
13. **主从复制**：首次 **全量复制**（主 bgsave 传 RDB + 期间写命令）；断线后 **增量复制**（靠复制积压缓冲区 `repl_backlog` + 复制偏移量 `offset`）。主从异步复制，存在数据不一致窗口。
14. **哨兵**：负责监控、故障转移、通知。**主观下线(SDOWN)** 单个哨兵判定 → 多个哨兵达到 **quorum** 判定 **客观下线(ODOWN)** → 用 **Raft** 选出 Leader 哨兵执行故障转移。
15. **Cluster**：去中心化，**16384 个哈希槽**，key 经 **CRC16 % 16384** 定位槽；节点间用 **Gossip** 协议交换状态；客户端访问错节点返回 **MOVED**（已迁移）或 **ASK**（迁移中）重定向。
16. **缓存三大问题**（面试必问，虽非本课重点也要会）：**缓存穿透**（查不存在的 key→布隆过滤器/缓存空值）、**缓存击穿**（热点 key 失效→互斥锁/逻辑过期）、**缓存雪崩**（大量 key 同时失效→过期时间加随机+多级缓存+高可用）。

---

## 三、核心流程图解（10 张图看透最难懂的机制）

> 纯文字最难懂的核心机制，用文本图直接看。

### 图1 · Redis 整体架构 ★ — 讲01

```
   客户端 ─┐
   客户端 ─┼─▶ IO多路复用(epoll) ─▶ 事件分发器 ─▶ [单线程] 命令处理
   客户端 ─┘        (监听大量连接)                   │
                                                     ▼
             全局哈希表 dict（键值空间 redisDb）
             key ─▶ redisObject{type, encoding, lru, refcount, *ptr}
                                                     │
             ┌───────────────┬──────────┬───────────┼──────────┐
           String          List        Hash        Set        ZSet
        int/embstr/raw  quicklist/   listpack/   intset/   listpack/
                        listpack     hashtable  hashtable  skiplist
                                                          +hashtable
   ┌─ 后台线程 BIO：异步关闭fd / AOF刷盘 / lazyfree删大key ─┐
```
**记忆点**：一个 redisObject 用 `type`（对外类型）+ `encoding`（底层编码）解耦——同一种类型底层可能是多种结构，按数据量自动切换（如 Hash 小用 listpack、大转 hashtable）。

### 图2 · 一条命令的处理流程（事件驱动）— 讲08~11

```
  ① 客户端连接 ─▶ acceptTcpHandler（连接应答处理器）
  ② 客户端发命令 ─▶ epoll 就绪 ─▶ readQueryFromClient（读+解析RESP协议）
  ③ 单线程执行命令 ─▶ 查/改全局 dict ─▶ 生成回复放入输出缓冲区
  ④ 可写事件就绪 ─▶ sendReplyToClient（回复客户端）
  ⑤ 时间事件 serverCron（每100ms）：过期删除采样/rehash推进/统计/RDB-AOF触发
```
**记忆点**：Redis 是 **单 Reactor 单线程** 模型。事件分两类：**文件事件**（连接/读/写，AE_READABLE/AE_WRITABLE）+ **时间事件**（serverCron 周期任务）。ae 框架封装了 select/poll/epoll/kqueue，编译时选最优（Linux 用 epoll）。

### 图3 · 为什么单线程还这么快 + 6.0 多线程 ★ — 讲12/13

```
  Redis 5.x 及以前（命令全程单线程）：
     [主线程]  读socket → 解析协议 → 执行命令 → 写socket   ← 全在一个线程

  Redis 6.0（多线程 IO，命令执行仍单线程）：
     [IO线程们]  读socket+解析 ─┐              ┌─ 写socket
                                ▼              │
     [主线程]                 执行命令(仍单线程)─┘
```
**记忆点**：单线程快的原因 = **纯内存 + IO多路复用 + 无锁无上下文切换 + 高效数据结构**。瓶颈其实在**网络 IO**，所以 6.0 让多个 IO 线程分担"读socket+协议解析"和"写socket"，但**命令执行始终单线程**（保证无需加锁、命令原子）。别答成"6.0 命令也多线程"。

### 图4 · 渐进式 rehash ★ — 讲03

```
   dict 有两个哈希表：ht[0]（旧）、ht[1]（新，扩容后更大）
   rehashidx = 当前迁移到的桶下标

   触发扩容(负载因子>1或强制) ─▶ 给 ht[1] 分配空间
        │
   每次增删改查操作，顺手把 ht[0][rehashidx] 这个桶的数据迁到 ht[1]，rehashidx++
        │
   期间：查询先查 ht[0] 再查 ht[1]；新增只写 ht[1]
        ▼
   迁移完毕 ─▶ ht[1] 变成 ht[0]，rehashidx=-1
```
**记忆点**：不一次性迁移（否则大表 rehash 会长时间阻塞主线程），而是**把迁移工作摊分到多次操作里**，用空间（两张表）换时间平滑。这是"渐进式"的精髓。

### 图5 · ZSet 双结构：skiplist + hashtable ★ — 讲05

```
   hashtable：  member ──▶ score        （O(1) 按成员查分数：ZSCORE）

   skiplist（跳表，按 score 排序，多层索引）：
     L3: head ───────────────▶ 50 ──────────▶ NULL
     L2: head ──────▶ 20 ────▶ 50 ────▶ 80 ──▶ NULL
     L1: head ▶10▶20▶30▶40▶50▶60▶70▶80▶90 ──▶ NULL
        （范围查询 ZRANGE / 排名 ZRANK：O(logN) 定位 + 顺链表遍历）
```
**记忆点**：两个结构存**同一份数据**、各取所长——hashtable 管**点查**（O(1)），skiplist 管**范围/排名查**（O(logN)）。跳表用"多层索引 + 抛硬币定层高"实现近似平衡，比红黑树好写、范围查询天然友好。

### 图6 · ziplist → quicklist → listpack 演进 — 讲06

```
  ziplist(压缩列表)：一整块连续内存存多个entry，省内存
      └─ 致命问题：连锁更新(cascade update)——某entry变长导致后续entry的
                   prevlen字段跟着变长，可能引发连续多次内存重分配

  quicklist：双向链表，每个节点是一个 ziplist（分段，限制单块大小）
      └─ 缓解连锁更新影响范围，兼顾内存与操作效率

  listpack：重新设计的紧凑列表，entry 不再存"前一项长度"
      └─ 从根源消除连锁更新（7.0 起全面替代 ziplist）
```
**记忆点**：三者都为"小数据省内存"服务。演进主线 = **解决 ziplist 的连锁更新问题**。listpack 每项只记录自身长度、不记前项长度，所以不会连锁。

### 图7 · RDB 快照：fork + 写时复制(COW) ★ — 讲18

```
  bgsave：
    主进程 ──fork()──▶ 子进程（共享同一份物理内存页）
      │                   │
   继续处理写命令       遍历内存，把数据写成 RDB 文件
      │                   │
   某页被主进程修改 ─▶ 触发写时复制COW：复制出一份新页给主进程改
                          （子进程仍看到 fork 那一刻的旧页 → 快照一致）
```
**记忆点**：**fork 子进程**来存盘，主进程照常服务；靠**写时复制 COW** 保证子进程看到的是 fork 瞬间的数据快照。代价：fork 本身在大内存实例上会短暂阻塞；写多时 COW 复制页会耗内存。

### 图8 · AOF 写后日志 + 重写 ★ — 讲19/20

```
  正常写入：执行命令 ──▶ 写 AOF 缓冲 ──▶ 按 appendfsync 落盘
            (先执行后记日志)          always/everysec(默认)/no

  AOF 重写(bgrewriteaof，AOF越来越大时)：
    主进程 ──fork──▶ 子进程：根据"当前内存数据"生成最小命令集 → 新AOF
      │
   重写期间的新写命令 ──▶ 同时写进「AOF重写缓冲区」
      │
   子进程写完 ─▶ 把重写缓冲区追加到新AOF ─▶ 替换旧AOF
```
**记忆点**：AOF 是**写后日志**（先执行命令、再记日志，好处：不阻塞当前写、不会记录错误命令）。重写不是读旧 AOF，而是**按当前数据生成等效的最小命令**。重写期间的新写靠**重写缓冲区**兜住，保证数据不丢。

### 图9 · 主从复制：全量 + 增量 ★ — 讲21

```
  首次全量复制：
    从 ──replicaof/psync──▶ 主
    主 bgsave 生成RDB ──传给从──▶ 从清空并加载RDB
    传输期间主的新写命令 ─存入─▶ 复制缓冲区 ─发给─▶ 从（追平）

  断线重连 → 增量复制：
    从带着 offset 重连 ─▶ 主查「复制积压缓冲区 repl_backlog」
       offset 还在缓冲区 ─▶ 只补发缺失部分（增量）
       offset 已被覆盖   ─▶ 退化为全量复制
```
**记忆点**：全量复制传 RDB（重），增量复制靠 **repl_backlog（环形缓冲区）+ offset（复制偏移量）+ runid**。积压缓冲区太小 → 断线久了就只能全量。主从是**异步复制**，故有主从不一致/丢数据窗口。

### 图10 · Cluster 哈希槽 + 重定向 ★ — 讲27

```
   16384 个哈希槽，分给各主节点：
     节点A: 槽 0~5460    节点B: 槽 5461~10922    节点C: 槽 10923~16383

   定位：slot = CRC16(key) % 16384 ─▶ 找到负责该槽的节点

   客户端访问到"不负责该槽"的节点：
     槽已完全迁移 ─▶ 返回 MOVED <slot> <newIP:port>（客户端更新缓存,以后直连）
     槽正在迁移中 ─▶ 返回 ASK <slot> <IP:port>（客户端本次临时重定向,加ASKING）
```
**记忆点**：Cluster **去中心化**，无代理，靠客户端重定向。**MOVED = 永久搬走**（更新本地槽映射），**ASK = 临时问一下**（迁移进行中，不更新映射）。集群状态靠 **Gossip** 协议（Ping/Pong/Meet/Fail）在节点间传播。

---

## 四、逐讲精要（数据结构模块 01-07）

## 01 | Redis 源码整体架构

**一句话核心**：Redis 是 C 语言实现的单进程、事件驱动的内存键值数据库，源码按"服务器实例 / 数据类型与操作 / 高可靠高可扩展 / 辅助功能"四条主线组织。

**关键知识点**
- 源码总目录四部分：`deps`（第三方库：hiredis 客户端、jemalloc 内存分配器、linenoise、lua）、`src`（核心功能，扁平单层目录，靠头文件互相 include）、`tests`（Tcl 编写的单测/集群/哨兵/主从测试）、`utils`（辅助脚本工具）。另有 `redis.conf`、`sentinel.conf`。
- 服务器实例主线：`server.c`（含 main 入口与主控流程）、事件驱动框架 `ae.c` + `ae_epoll/kqueue/select/evport`、TCP 封装 `anet.c`、客户端 `networking.c`。
- 数据类型主线：五种基本类型 String/List/Hash/Set/Sorted Set，底层数据结构多样（SDS、dict、ziplist、intset、skiplist、quicklist、listpack、rax）；键值对增删改查在 `db.c`；`object.c` 定义 redisObject。
- 内存优化三方面：内存分配（`zmalloc.c` 封装 jemalloc/tcmalloc）、内存回收（`expire.c` 过期、`lazyfree.c` 异步删除）、数据替换（`evict.c` LRU/LFU）。
- 高可靠/高可扩展：持久化 `rdb.c`、`aof.c`；主从复制 `replication.c`；哨兵 `sentinel.c`；集群 `cluster.c`。
- 后台异步任务在 `bio.c`：4.0 起有 3 类后台线程 BIO_CLOSE_FILE（异步关文件）、BIO_AOF_FSYNC（AOF 刷盘）、BIO_LAZY_FREE（异步释放内存）；主线程通过 `bioCreateBackgroundJob` 发布任务。

**面试高频问答**
- **Q：** Redis 用什么数据结构保存所有键值对？
  **A：** 一个全局哈希表（dict），键值空间就是这个 dict，查询理论 O(1)；每个 DB 还有一个 expires dict 存过期时间。
- **Q：** Redis 4.0 新增的后台异步任务用来做什么？
  **A：** 把耗时操作从主线程剥离，交由 bio 后台线程做，典型是 UNLINK/异步删除大 key（BIO_LAZY_FREE），避免释放大对象阻塞主线程；还有关文件、AOF fsync。

---

## 02 | 字符串实现：为什么用 SDS 而不是 char*

**一句话核心**：Redis 自定义 SDS（简单动态字符串），用 len/alloc 元数据换来 O(1) 取长度、二进制安全、避免缓冲区溢出和频繁重分配。

**关键知识点**
- char* 的两大不足：① 以 `\0` 标识结尾 → 二进制不安全，遇到 `\0` 数据被截断；② 取长度 strlen 要遍历 O(N)，追加 strcat 需手动保证空间、易溢出。
- SDS 结构：字符数组 `buf[]` + 元数据 `len`（现有长度）、`alloc`（已分配空间，不含头和 `\0`）、`flags`（类型）。`typedef char *sds`，本质仍是字符数组，对外可像 char* 使用。
- 二进制安全：靠 len 判断长度而非 `\0`，可存图片等任意二进制。
- 高效操作：取长度直接读 len，O(1)；追加 `sdscatlen` 先调 `sdsMakeRoomFor` 检查扩容再拷贝，把空间检查/扩容封装，避免溢出。
- 内存预分配（减少重分配）：扩容时多分配——小于 1MB 翻倍，大于 1MB 每次多加 1MB；缩容不立即释放（惰性），供下次复用。
- 5 种类型 sdshdr5/8/16/32/64，区别在 len 和 alloc 的整型宽度（uint8/16/32/64），按字符串大小选最省内存的头，避免小串用大头浪费。
- `__attribute__((__packed__))` 取消字节对齐，紧凑分配内存，进一步省空间。

**面试高频问答**
- **Q：** SDS 相比 C 字符串的优势？
  **A：** ① O(1) 取长度；② 二进制安全（可存 `\0`）；③ 杜绝缓冲区溢出（追加前自动扩容）；④ 空间预分配 + 惰性释放，减少内存重分配次数；⑤ 兼容部分 C 字符串函数（仍以 `\0` 结尾）。
- **Q：** SDS 为什么设计 5 种 header 类型？
  **A：** 不同大小字符串用不同宽度的 len/alloc，短串配小 header，节省元数据内存开销。
- **Q：** SDS 在 Redis 里哪些地方被用到？
  **A：** 所有 key、client 的输入缓冲区 querybuf、AOF 缓冲区 aof_buf、集群节点通信缓冲区等。

---

## 03 | 如何实现性能优异的 Hash 表（dict）

**一句话核心**：dict 用链式哈希解决冲突，用两张表 + 渐进式 rehash 把扩容开销平摊到每次操作，避免一次性迁移阻塞主线程。

**关键知识点**
- 结构：`dict` 内含 `dictht ht[2]`（两张哈希表，交替用于 rehash）+ `rehashidx`（-1 表示未 rehash，否则记录当前迁移到第几个 bucket）。`dictht` 是 dictEntry 指针数组，size/used/sizemask。
- 链式哈希：`dictEntry` 有 `next` 指针，冲突的 key 用链表串起来。value 用联合体 v（可存指针，也可直接存 int64/double），整数/浮点直接内联省一个指针。
- rehash 基本流程：正常写入 ht[0]；rehash 时数据迁到 ht[1]；迁完释放 ht[0]、把 ht[1] 赋给 ht[0]、ht[1] 置空，rehashidx 置 -1。
- 触发条件（负载因子 load factor = used/size）：① ht[0] 大小为 0 → 初始化；② load factor ≥ 1 且允许 resize（无 RDB/AOF 子进程时才允许，`updateDictResizePolicy`）；③ load factor ≥ 5（dict_force_resize_ratio）强制扩容，无视是否有子进程。
- 扩容大小：扩到 used*2 向上取最近的 2 的幂（`_dictNextPower`）。
- 渐进式 rehash：不一次迁完，每次操作只迁一个 bucket（`_dictRehashStep` → `dictRehash(d,1)`），把开销平摊；增删查（dictAddRaw/dictGenericDelete/dictFind 等 5 个函数）都会触发。另有定时 rehash：主线程每 100ms 执行一次，每次迁 100 桶、限时 1ms。
- rehash 期间查询要先查 ht[0]，找不到再查 ht[1]；新增只写 ht[1]。
- 缩容：使用率不足 10%（HASHTABLE_MIN_FILL）触发，同样走渐进式。
- 哈希函数：4.0 起用 SipHash（抗哈希洪水攻击），3.0-4.0 是 MurmurHash2。

**面试高频问答**
- **Q：** 什么是渐进式 rehash，为什么需要它？
  **A：** rehash 需把所有 key 从旧表拷到新表，一次性做会长时间阻塞单线程主线程。渐进式把迁移分批：每次增删查操作顺带迁一个 bucket，配合定时任务，用两张表并存过渡，直到迁完，避免卡顿。
- **Q：** rehash 触发条件？为什么有 RDB/AOF 时通常不 rehash？
  **A：** 负载因子≥1 且允许 resize 时扩容，≥5 强制扩容。有 RDB/AOF 子进程时，子进程与父进程共享内存（写时复制 COW），rehash 会大量修改内存导致大量页复制，所以暂时禁用（除非负载因子已≥5 太严重）。
- **Q：** dict 为什么不像 Java HashMap 那样用红黑树？
  **A：** Redis 追求高性能且冲突通常不严重，链表在低负载下 O(1) 优于红黑树；链表↔红黑树转换有额外开销和空间浪费；渐进式 rehash 已能及时扩容控制链长。

---

## 04 | 内存友好的数据结构设计

**一句话核心**：Redis 靠"连续内存 + 变长元数据编码"节省内存，代表是 embstr 嵌入式字符串、ziplist、intset，再加共享对象减少冗余。

**关键知识点**
- redisObject（robj）：`type`（面向用户类型 String/List/...）、`encoding`（底层编码 SDS/ziplist/intset/skiplist...）、`lru`、`refcount`、`ptr`。是连接"上层类型"与"底层结构"的桥梁。
- 位域定义：type、encoding 各占 4bit，lru 占 24bit，共用一个 unsigned 的字节，节省内存（C 位域技巧）。
- 嵌入式字符串 embstr：String ≤ 44 字节时，redisObject 与 SDS 分配在一块连续内存里（`createEmbeddedStringObject`），只分配一次内存、无内存碎片；> 44 字节用 raw（`createRawStringObject`），robj 和 SDS 分开分配两次。
- 44 字节由来：jemalloc 一次能放进 64 字节 arena；64 - 16(robj) - 3(sdshdr8) - 1(`\0`) = 44。
- ziplist（压缩列表）：一整块连续内存，无结构体定义。头部含总字节数、尾偏移、元素个数（共 10 字节），尾部 1 字节 ZIP_END(255)。列表项 = prevlen（前项长度）+ encoding（当前项编码）+ data。用变长编码（prevlen 1 或 5 字节；encoding 按整数/字符串长度分档）省内存。缺点：查中间元素需遍历、有连锁更新风险。
- intset（整数集合）：Set 全是整数时使用，连续内存的有序数组 contents，支持二分查找；按数值范围用 int16/int32/int64 编码，会升级/降级。
- 共享对象：把常用小整数（0~9999）和常见回复（+OK、-ERR 等）预创建为共享 redisObject，只读复用，避免重复创建。仅适用于只读场景。

**面试高频问答**
- **Q：** String 底层有哪几种编码，如何选择？
  **A：** int（整数且能用 long 表示，≤10000 走共享对象池）、embstr（≤44 字节，robj+SDS 一次分配）、raw（>44 字节，分两次分配）。
- **Q：** ziplist 为什么省内存又有什么代价？
  **A：** 连续内存无指针开销、变长编码。代价是查中间元素 O(N)、元素多或变大时重分配内存，且 prevlen 变长可能引发连锁更新。

---

## 05 | 有序集合 Sorted Set：为何同时支持点查与范围查

**一句话核心**：zset = 跳表(skiplist) + 哈希表(dict) 双索引，跳表支持 O(logN) 范围查询，哈希表支持 O(1) 按成员取权重。

**关键知识点**
- 结构 `zset { dict *dict; zskiplist *zsl; }`：dict 存 member→score（ZSCORE O(1)），跳表按 score 有序（ZRANGEBYSCORE O(logN)+M）。
- 跳表节点 `zskiplistNode`：ele(sds)、score(double)、backward（后向指针，支持倒序）、level[] 数组（每层含 forward 前向指针 + span 跨度）。
- span 跨度：记录该层前向指针跨越了底层多少节点，累加可算出节点排名，用于 ZRANK/ZREVRANK。
- 查询：从最高层头节点开始，先比 score 再比 ele，比目标小就在本层前进，否则下沉一层；层高越高节点越少跨度越大，类似二分，O(logN)。
- 层数随机确定：`zslRandomLevel`，初始 1 层，每次以 ZSKIPLIST_P=0.25 概率 +1 层，最大 64 层。随机层数使插入只需改前后指针、无需为维持严格 2:1 比例而调整其他节点，避免连锁更新。
- 数据一致性：`zsetAdd` 中先 dictFind 查是否存在，新增则分别 zslInsert（跳表）+ dictAdd（哈希表）；改权重则 zslUpdateScore 后让 dict 的 value 指向跳表节点的 score。跳表与哈希表操作保持独立、依次执行。

**面试高频问答**
- **Q：** zset 为什么同时用跳表和哈希表？
  **A：** 单一结构难兼顾。哈希表按 member O(1) 取 score（ZSCORE），跳表维护 score 有序、支持 O(logN) 范围查询和 rank（ZRANGE）。双索引各取所长，代价是空间换时间。
- **Q：** 为什么用跳表而不是红黑树/平衡树？
  **A：** ① 更省内存：0.25 概率下平均每节点约 1.33 个指针，平衡树固定 2 个；② 范围查询友好：跳表底层是有序链表，定位后直接顺着链表遍历，平衡树要中序遍历；③ 实现简单、插入只改前后指针，无需再平衡。
- **Q：** 双索引有什么不足？
  **A：** 空间换时间，内存占用更高；dict 扩容/大范围删除时要同步维护两个结构，删除还要同步 dict、可能触发 rehash，开销较大。

---

## 06 | 从 ziplist 到 quicklist 再到 listpack

**一句话核心**：ziplist 因每项记录前项长度存在连锁更新缺陷；quicklist 用"链表串多个 ziplist"限制影响范围，listpack 通过不再记录前项长度从根本上消除连锁更新。

**关键知识点**
- ziplist 两大不足：① 查中间元素需从头/尾遍历，元素多时性能降；② 连锁更新——每项存 prevlen（前项长度），前项变大导致本项 prevlen 从 1 字节涨到 5 字节，进而后续项都要跟着扩，级联重分配内存，性能骤降。
- 插入流程（`__ziplistInsert`）：算插入元素长度（整数/字符串不同编码）→ 算 prevlen/encoding → `zipPrevLenByteDiff` 判断后项 prevlen 空间差 nextdiff → `ziplistResize`(=curlen+reqlen+nextdiff) 调 zrealloc 重分配。
- quicklist（3.0）：一个链表，每个节点 quicklistNode 是一个 ziplist。节点含 prev/next 指针、`*zl` 指向 ziplist、sz、count、encoding 等。插入前 `_quicklistNodeAllowInsert` 判断该 ziplist 是否超限（默认单 ziplist ≤ 8KB 或元素个数受 list-max-ziplist-size 控制），超了就新建节点。把连锁更新影响限制在单个 ziplist 内。缺点：quicklistNode 结构本身有额外内存开销。
- listpack（5.0，紧凑列表）：连续内存。头 6 字节（4 字节总字节数 + 2 字节元素数），尾 1 字节 LP_EOF(255)。列表项 = encoding（编码类型）+ data（数据）+ **entry-len**（前两部分的长度，放在项末尾）。**不再记录前项长度**，故修改某项不影响其他项 → 彻底消除连锁更新。
- listpack 变长编码：整数 7BIT_UINT / 13/16/24/32/64BIT_INT，字符串 6/12/32BIT_STR，用编码首字节高位标识类型。
- listpack 双向遍历：正向靠 lpFirst/lpNext（跳过项长度）；反向靠 entry-len——每字节最高位标识是否结束、低 7 位存长度且采用大端存储，从右往左解析出前项长度即可回退。
- 目前只有 Stream 用了 listpack；List/Hash/Set/ZSet 仍依赖 ziplist（替换是渐进过程）。List 底层就是 quicklist，两端操作 O(1)。

**面试高频问答**
- **Q：** 什么是 ziplist 连锁更新？
  **A：** 每个列表项存前一项长度 prevlen（1 或 5 字节）。当某项变大越过 254 字节阈值，后一项的 prevlen 要从 1 字节扩到 5 字节，本项变大又触发再后一项扩容，级联下去多项都重分配内存，最坏 O(N²)，性能骤降。
- **Q：** quicklist 和 listpack 分别怎么解决 ziplist 问题？
  **A：** quicklist 把数据拆到多个 ziplist 节点、限制单节点大小，连锁更新最多影响一个 ziplist；listpack 改结构——每项只存自己的长度不存前项长度，从根本上消除连锁更新。
- **Q：** listpack 不记录前项长度，怎么反向遍历？
  **A：** 每项末尾有 entry-len（存本项 encoding+data 的长度），用高位标识 + 大端 7 位编码，从右往左逐字节解析出前项长度即可回退到前一项。

---

## 07 | 为什么 Stream 使用 Radix Tree（基数树）

**一句话核心**：Stream 消息 ID 有大量公共前缀，用 Radix Tree 存 ID（省内存、有序可范围查），用 listpack 存消息内容作为 value。

**关键知识点**
- Stream 消息特征：一条消息含消息 ID（时间戳+序号，自动生成且递增）+ 一或多个键值对；连续消息 ID 前缀高度重复、键名也常相同。若用哈希表存会有大量冗余前缀浪费内存。
- Radix Tree 是前缀树（Trie）的压缩优化版：前缀树每节点存单字符、共享公共前缀省内存；但单链分支也拆成单字符会浪费空间、降查询效率。Radix Tree 把"唯一分支的连续单字符节点"合并成一个压缩节点。
- 两类节点：① 非压缩节点——含多个子节点指针 + 各子节点对应的单字符（如 "r"→a、e）；② 压缩节点——含 1 个子节点指针 + 合并字符串（如 "dis"）。
- raxNode 结构：`iskey`（根到本节点路径是否构成完整 key，不含本节点内容）、`isnull`（是否空节点/无 value）、`iscompr`（是否压缩节点）、`size`（压缩节点=字符串长度，非压缩节点=子节点数）、`data[]`（子节点字符/合并串 + 子节点指针 + value 指针）。叶子节点 size=0、无子指针，若是 key 则存 value 指针。为内存对齐会 padding。
- 关键函数：raxNew（建树）、raxNewNode（建非压缩节点）、raxCompressNode（建压缩节点）、raxGenericInsert（插入）、raxLowWalk（查找/插入/删除时遍历）、raxGetData/raxSetData。
- Stream 组合：`stream{ rax *rax; length; last_id; rax *cgroups; }`。消息 ID 作 Radix Tree 的 key，消息内容存 listpack 作 value（raxNode 的 value 指针指向 listpack）；listpack 用 master entry 存共同的键名，只存一份，进一步省内存。

**面试高频问答**
- **Q：** Stream 为什么用 Radix Tree + listpack？
  **A：** 消息 ID 前缀高度重复，Radix Tree 共享公共前缀极省内存，且本身有序、支持消息 ID 的单点与范围查询；消息内容用紧凑的 listpack 存储，键名共享一份，整体最省内存。所以推荐用默认自动生成的 ID 以发挥公共前缀优势。
- **Q：** Radix Tree 相比前缀树的改进？
  **A：** 前缀树每节点只存单字符，唯一分支的一串单字符节点会浪费空间、增加查询跳数。Radix Tree 把这种唯一分支的连续节点压缩合并成压缩节点，既省内存又减少查询时的节点匹配次数。
- **Q：** Radix Tree 对比 B+ 树、跳表的优劣？
  **A：** 优势：存公共前缀数据更省内存，查询复杂度 O(K) 只与 key 长度相关而与数据量无关，适合前缀匹配/自动补全。不足：公共前缀少时反而更费内存，增删要处理节点分裂/合并（跳表只改前后指针），范围查询不如 B+树/跳表直接遍历链表友好，实现更复杂。

---

## 五、逐讲精要（事件驱动与执行模型 08-14）

## 08 | Redis server 启动流程

**一句话核心**：main 函数五阶段启动 Redis，核心是「解析配置 → initServer 初始化 → aeMain 进入事件循环」。

**关键知识点**
- main 函数（server.c）五阶段：①基本初始化（时区、哈希随机种子）；②检查哨兵模式 / RDB/AOF 检测；③运行参数解析；④initServer 初始化；⑤aeMain 执行事件驱动框架。
- 参数三轮赋值：`initServerConfig` 设默认值（server.h 中 `CONFIG_DEFAULT_*` 宏）→ 解析命令行参数（命令行用双减号 `--`）→ `loadServerConfig`/`loadServerConfigFromString` 读配置文件并合并覆盖。默认端口 6379，后台任务频率 hz 默认 10。
- initServer 三步：①初始化资源管理结构（客户端链表、从库链表、淘汰候选池、状态统计）；②初始化数据库（循环为每个 db 创建全局哈希表 dict、expires 过期表、blocking_keys、watched_keys 等）；③创建事件驱动框架并监听端口。
- 事件框架关键调用：`aeCreateEventLoop` 创建事件循环 → `listenToPort` 监听端口 → `aeCreateTimeEvent` 注册时间事件 serverCron → `aeCreateFileEvent` 为监听 fd 注册 AE_READABLE 事件，处理函数 `acceptTcpHandler`。
- 最后 main 调用 `aeMain` 进入事件循环，之前用 `aeSetBeforeSleepProc`/`aeSetAfterSleepProc` 设置 beforeSleep/afterSleep 钩子。
- 数据加载：`loadDataFromDisk` 先读 AOF，没有 AOF 才读 RDB（AOF 通常更完整、数据更新）。
- 补充：initServer 之后还会调用 `InitServerLast` 启动 3 个 BIO 后台线程。

**面试高频问答**
- **Q：** Redis 启动都做了哪些事？
  **A：** 基本初始化 → 设默认配置（initServerConfig）→ 加载命令行/配置文件参数（loadServerConfig）→ initServer 初始化资源、数据库、事件框架并监听端口、注册 serverCron 时间事件 → 启动 BIO 后台线程 → aeMain 进入事件循环处理请求。
- **Q：** Redis 参数是怎么设置和覆盖的？
  **A：** 三轮赋值，优先级从低到高：默认值（CONFIG_DEFAULT 宏）< 配置文件 < 命令行参数，后者覆盖前者。
- **Q：** Redis 启动先加载 RDB 还是 AOF？
  **A：** 若开启 AOF 则优先加载 AOF，没有 AOF 才加载 RDB，因为 AOF 记录更完整、丢数据更少。

---

## 09 | IO 多路复用：select / poll / epoll

**一句话核心**：Redis 单线程靠 IO 多路复用同时监听大量 socket，Linux 上首选性能最优的 epoll。

**关键知识点**
- 基本 Socket 模型（socket→bind→listen→accept→recv/send）一次只能处理一个连接；多线程方案对单线程的 Redis 不适用，因此用 IO 多路复用：一个线程同时监听多个 fd，有就绪的就返回处理。
- 评估一种多路复用机制看三点：监听哪些事件、能监听多少 fd、如何找到就绪 fd。
- **select**：用 3 个 fd_set（读/写/异常），单进程最多监听 1024 个 fd（FD_SETSIZE）；返回后需遍历全部 fd 找就绪的（O(n)）；每次调用要把 fd 集合从用户态拷到内核态，且 fd_set 会被内核修改不可重用。
- **poll**：用 pollfd 数组代替位图，突破了 1024 上限；pollfd 可重用（revents 独立字段）。但仍需遍历所有 fd、每次仍要拷贝，本质缺陷同 select。
- **epoll**：`epoll_create` 建实例（内部红黑树存监听 fd + 就绪链表），`epoll_ctl` 增删改监听事件，`epoll_wait` 直接返回已就绪 fd。无需遍历全部 fd、fd 数量无硬限制、内核回调（ep_poll_callback）把就绪 fd 放入就绪链表，效率最高。
- Redis 事件框架抽象层对不同 OS 封装：ae_epoll.c（Linux）、ae_kqueue.c（macOS/FreeBSD）、ae_evport.c（Solaris）、ae_select.c（Windows/兜底），编译期按宏选择。

**面试高频问答**
- **Q：** select、poll、epoll 有什么区别？
  **A：** ①fd 数量：select 限 1024，poll 无限制，epoll 无限制；②查找就绪：select/poll 需遍历全部 fd（O(n)），epoll 直接返回就绪 fd；③拷贝开销：select/poll 每次调用都要把 fd 集合从用户态拷到内核态，epoll 用红黑树在内核维护、仅注册时拷贝；④epoll 靠回调把就绪 fd 放入链表，性能不随连接数下降。
- **Q：** 为什么 epoll 比 select/poll 高效？
  **A：** 内核用红黑树管理 fd（增删改 O(logn)）、通过设备回调 ep_poll_callback 主动把就绪 fd 加入就绪链表，epoll_wait 直接取就绪链表，避免了每次全量遍历和拷贝。
- **Q：** Redis 为什么没用 poll？
  **A：** Linux 上 epoll 性能优于 poll，Redis 直接用 epoll；Windows 不支持 epoll/poll 则用 select。poll 相对 select 只解决了 fd 上限，仍要遍历，没有明显优势，故未采用（rewriteaof 中的 aeWait 才用到 poll 做短暂阻塞）。

---

## 10 | 事件驱动框架（中）：Redis 是 Reactor 模型吗？

**一句话核心**：是。Redis 基于单 Reactor 单线程模型自研事件驱动框架（ae.c），用一个循环完成事件的监听、分发、处理。

**关键知识点**
- Reactor 模型「两个三」：三类事件（连接、读、写）+ 三个角色（reactor 监听分发、acceptor 接收连接、handler 处理读写）。
- Reactor 三种变体：单 Reactor 单线程（Redis 6.0 前）、单 Reactor 多线程、主从 Reactor 多线程（如 Netty）。Redis 6.0 前属单 Reactor 单线程：监听、读取、命令执行、写回都在一个线程。
- 事件驱动框架 = 事件初始化 + 事件捕获/分发/处理主循环（while 循环）。
- Redis 定义两类事件：IO 事件（aeFileEvent，对应网络请求）和时间事件（aeTimeEvent，对应周期任务）。
- 三个关键函数：`aeMain`（主循环，不停调 aeProcessEvents）、`aeProcessEvents`（捕获+分发，内部调 aeApiPoll）、`aeCreateFileEvent`（注册事件与 handler，底层调 aeApiAddEvent→epoll_ctl）。
- `aeApiPoll` 是抽象封装，Linux 上即封装 `epoll_wait`，捕获就绪事件后按类型调用注册的回调函数。
- aeFileEvent 结构：mask（AE_READABLE/AE_WRITABLE/AE_BARRIER）、rfileProc/wfileProc 两个回调、clientData。

**面试高频问答**
- **Q：** Redis 实现了 Reactor 模型吗？（经典题）
  **A：** 实现了。答两部分：①Reactor 三类事件（连接/读/写）+ 三角色（reactor/acceptor/handler）；②Redis 用 ae.c 自研事件驱动框架对应：reactor=aeMain+aeProcessEvents 主循环，acceptor=acceptTcpHandler，handler=readQueryFromClient/sendReplyToClient；底层用 epoll。Redis 6.0 前是单 Reactor 单线程。
- **Q：** aeApiPoll 的作用？
  **A：** 事件框架对操作系统 IO 多路复用函数的统一封装，Linux 下封装 epoll_wait，用于捕获一批就绪的 fd 事件返回给主循环分发处理。
- **Q：** 还有哪些系统用了 Reactor？
  **A：** Netty（主从多 Reactor）、Nginx（多进程，master 不处理 IO、worker 各自单 Reactor）、Kafka、Memcached。

---

## 11 | 事件驱动框架（下）：Redis 有哪些事件？

**一句话核心**：Redis 事件分 IO 事件（可读/可写/屏障）和时间事件（serverCron），一个循环内统一处理。

**关键知识点**
- aeEventLoop 结构记录：events（IO 事件数组，按 fd 索引）、timeEventHead（时间事件链表头）、fired（已触发事件数组）、apidata、beforesleep/aftersleep 钩子。
- `aeCreateEventLoop(setsize)` 初始化：setsize = maxclients + CONFIG_FDSET_INCR，决定能监听的最大客户端数；调 aeApiCreate（Linux 下 epoll_create + 分配 epoll_event 数组）；把所有 fd 掩码初始化为 AE_NONE。客户端报「max number of clients reached」就改 redis.conf 的 maxclients。
- **IO 事件三类**：可读、可写、屏障（AE_BARRIER，用于反转处理顺序——正常先回复客户端，需先落盘时先写数据再回复）。
- **连接事件**：监听 fd 上注册 AE_READABLE，回调 `acceptTcpHandler` → acceptCommonHandler → createClient，在新的已连接 fd 上再注册 AE_READABLE，回调 `readQueryFromClient`。
- **读事件**：无论客户端发读还是写请求，对 server 都是「读取并解析请求」，故已连接 socket 上注册的都是可读事件，回调 readQueryFromClient。
- **写事件**：命令处理后数据先写入客户端输出缓冲区；beforeSleep 中调 `handleClientsWithPendingWrites` 遍历待写客户端调 writeToClient 写回；若一次没写完，注册 AE_WRITABLE 事件，回调 `sendReplyToClient`。
- **时间事件**：`aeCreateTimeEvent` 创建，回调 `serverCron`；以链表组织；aeProcessEvents 末尾调 processTimeEvents 检查到时事件执行。serverCron 里用 `run_with_period` 宏按 hz 控制不同频率的周期任务（如每秒检查 AOF 写错误、过期 key 清理、rehash 等）。

**面试高频问答**
- **Q：** Redis 事件循环处理哪些事件？
  **A：** 两类：文件（IO）事件——新连接（acceptTcpHandler）、读请求（readQueryFromClient）、写回响应（sendReplyToClient）；时间事件——周期性定时任务 serverCron（过期清理、rehash、AOF 检查、统计等），都在主线程一个循环内处理。
- **Q：** 客户端发的是写命令，为什么 server 注册的是可读事件？
  **A：** 对 server 而言，无论客户端发读还是写命令，都要先「读取并解析」客户端发来的请求内容，所以已连接 socket 上监听的是可读事件。
- **Q：** 一个循环里怎么同时处理 IO 事件和定时任务？
  **A：** aeApiPoll 的超时时间设为最近时间事件的到期时间，这样即使没有 IO 事件，epoll_wait 也会按时返回，随后 processTimeEvents 执行 serverCron，实现单线程内 IO + 定时任务并存。

---

## 12 | Redis 真的是单线程吗？

**一句话核心**：命令处理（接收/解析/执行/读写）是单线程（主 IO 线程），但另有 3 个 BIO 后台线程异步处理耗时操作，所以整体不是纯单线程。

**关键知识点**
- 启动：shell `fork` 子进程 → `execve` 替换为 redis-server → 执行 main。守护进程由 `daemonize` 实现：fork 后父进程 exit(0)，子进程 setsid 创建新会话、输入输出重定向到 /dev/null。
- fork 返回值：<0 出错；=0 子进程分支；>0 父进程分支（返回值是子进程 PID）。
- **为什么单线程还快**：①纯内存操作，瓶颈不在 CPU 而在网络 IO；②IO 多路复用高效处理并发连接；③单线程避免了多线程锁竞争和上下文切换开销；④高效的数据结构。
- **单线程的缺点**：任一请求耗时（bigkey、一次取数据过多）会阻塞后续请求排队，延迟升高。
- Redis 3.0 后引入 BIO 后台线程（bio.c），把耗时操作从主线程剥离，避免阻塞主 IO 线程。
- **3 个 BIO 后台线程**（BIO_NUM_OPS=3）：BIO_CLOSE_FILE（异步关闭文件 fd）、BIO_AOF_FSYNC（AOF 异步刷盘 fsync）、BIO_LAZY_FREE（惰性删除/异步释放内存）。
- BIO 机制是**生产者-消费者模型**：`bioCreateBackgroundJob`（生产者）把任务加入 bio_jobs 队列；`bioProcessBackgroundJobs`（消费者）后台线程 while 循环轮询队列取任务执行；`bioInit`（由 InitServerLast 调用）用 pthread_create 创建 3 个线程。

**面试高频问答**
- **Q：** Redis 是单线程吗？
  **A：** 不完全准确。处理客户端请求的核心逻辑（接收、解析、执行命令、读写数据）是单线程（主 IO 线程），这是「Redis 单线程」的由来；但从 3.0 起还有 3 个 BIO 后台线程做关闭 fd、AOF fsync、lazyfree 等耗时操作，6.0 又加了多 IO 线程，所以整个 Server 是多线程的。
- **Q：** Redis 单线程为什么这么快？
  **A：** 纯内存操作（瓶颈在网络 IO 不在 CPU）+ IO 多路复用 + 单线程避免锁和上下文切换 + 高效数据结构。
- **Q：** 哪些操作放到后台线程了？为什么？
  **A：** 关闭文件（close）、AOF 刷盘（fsync）、释放大对象内存（lazyfree）三类耗时操作，避免它们阻塞主线程影响请求处理延迟；采用生产者-消费者队列，主线程投递任务、后台线程消费执行。

---

## 13 | Redis 6.0 多 IO 线程

**一句话核心**：6.0 引入多 IO 线程并行处理网络读写和命令解析，但命令执行仍是主线程单线程，用于加速网络 IO 而非命令执行。

**关键知识点**
- 初始化：`InitServerLast` 在 bioInit 后调 `initThreadedIO`；按配置 `io-threads` N 创建线程：io_threads_num=1 则退化为纯单线程；线程编号 0 是主 IO 线程，编号 1~N-1 用 pthread_create 创建，运行函数 `IOThreadMain`；上限 IO_THREADS_MAX_NUM=128。
- 四个数组：io_threads_list（每线程待处理客户端列表）、io_threads_pending（待处理客户端数）、io_threads_mutex（互斥锁）、io_threads（线程描述符）。
- `IOThreadMain`：while(1) 循环，从 io_threads_list 取客户端，按 io_threads_op 标记决定操作——IO_THREADS_OP_READ 调 readQueryFromClient 读，IO_THREADS_OP_WRITE 调 writeToClient 写。
- **推迟读**：readQueryFromClient 调 `postponeClientRead`，满足条件（IO 线程激活、io-threads-do-reads=yes、非阻塞事件处理中、非主从客户端）就打 CLIENT_PENDING_READ 标记，加入 `clients_pending_read`。默认 io-threads-do-reads=no，即默认不用多线程读。
- **推迟写**：addReply → prepareClientToWrite → clientInstallWriteHandler，打 CLIENT_PENDING_WRITE 标记，加入 `clients_pending_write`。
- **分配**：beforeSleep 中 `handleClientsWithPendingReadsUsingThreads` / `handleClientsWithPendingWritesUsingThreads` 把两个列表的客户端按**轮询方式（序号对线程数取模）**分配到 io_threads_list；主线程（0 号）也处理一份，然后 while 自旋等待所有 IO 线程完成，再由主线程串行执行后续。
- **写优化**：待写客户端数 < IO 线程数×2 时，不启多线程，直接主线程 handleClientsWithPendingWrites 处理，节省 CPU。
- **核心结论**：多 IO 线程只负责读取数据、解析命令、写回数据，**命令执行始终在主 IO 线程**。避免 bigkey、避免阻塞操作等原有优化仍然有效。
- 官方建议：≥4 核才开，4 核开 2-3 个、8 核开 6 个，超过 8 线程收益不大；开启后吞吐约翻倍。

**面试高频问答**
- **Q：** Redis 6.0 多线程做什么？命令执行是多线程吗？
  **A：** 多 IO 线程只并行处理网络 IO——读取 socket 数据、解析协议命令、写回响应；命令的实际执行仍由主 IO 线程单线程完成。目的是利用多核加速网络 IO（6.0 前网络读写是瓶颈），不改变命令串行执行的语义。
- **Q：** 6.0 多线程怎么工作？
  **A：** 启动按 io-threads 建线程；读写时把客户端推迟加入 clients_pending_read/write 列表；进入事件循环前 beforeSleep 把客户端轮询分配给各 IO 线程的 io_threads_list；IO 线程并行读/写，主线程等所有线程完成后串行执行命令。默认只多线程写，多线程读需开 io-threads-do-reads=yes。
- **Q：** 为什么命令执行还坚持单线程？
  **A：** 保持命令串行、天然原子，避免多线程操作同一 key 的加锁开销和竞争；且瓶颈主要在网络 IO，多线程化网络读写即可获得主要收益。

---

## 14 | 从命令执行看分布式锁的原子性

**一句话核心**：加锁用 `SET key uid EX time NX`、解锁用 Lua 脚本，原子性靠命令执行始终在主线程单线程串行执行来保证，IO 多路复用和多 IO 线程都不破坏它。

**关键知识点**
- **加锁**：`SET lockKey uid EX expireTime NX`。NX 保证 key 不存在才创建（已存在返回 NULL，加锁失败）实现互斥；EX 设过期时间避免死锁；uid 是客户端唯一标识用于安全解锁。SET+NX+EX 是一条命令，原子完成，替代早期 SETNX+EXPIRE 两条命令的非原子问题。
- **解锁**：用 Lua 脚本经 EVAL 执行——先 GET 判断 value 是否等于自己的 uid，相等才 DEL，防止误删别人的锁。判断+删除需放在 Lua 脚本里保证原子。
- **命令处理四阶段**：①读取 readQueryFromClient；②解析 processInputBuffer(AndReplicate)（按开头是否 `*` 分 RESP 协议 processMultibulkBuffer / 内联 processInlineBuffer）；③执行 processCommand（lookupCommand 查 commands 哈希表 → call → 具体命令函数如 setCommand→setGenericCommand）；④返回 addReply（写入输出缓冲区 + 加入 clients_pending_write）。
- SET 执行：setGenericCommand 中若带 NX 且 lookupKeyWrite 发现 key 已存在，直接 addReply 返回空值（符合加锁失败语义）；否则 setKey 插入、setExpire 设过期、addReply 返回 OK。
- **IO 多路复用不破坏原子性**：aeApiPoll 一次拿到多个就绪 fd，但主线程仍是 for 循环逐一调用回调处理，命令逐个串行执行。
- **多 IO 线程不破坏原子性**：IO 线程只做读取和解析第一个命令（打 CLIENT_PENDING_COMMAND 标记后退出，不执行）；命令的实际执行由主线程的 handleClientsWithPendingReadsUsingThreads 调 processCommandAndResetClient 串行完成；写回阶段命令早已执行完，多线程并发写只是返回结果，不影响执行结果。
- **结论**：命令执行永远单线程串行 → 天然原子 → 分布式锁原子性得到保证。多 IO 线程不加速命令执行，若命令执行成瓶颈应上切片集群。

**面试高频问答**
- **Q：** 用 Redis 怎么实现分布式锁？
  **A：** 加锁 `SET lockKey uid EX time NX`：NX 保证互斥、EX 设过期防死锁、uid 标识持有者。解锁用 Lua 脚本：GET 比对 value==自己 uid 才 DEL，防误删他人锁。
- **Q：** 为什么加锁要用一条 SET NX EX 而不是 SETNX + EXPIRE？
  **A：** 两条命令非原子，若 SETNX 后、EXPIRE 前客户端崩溃，key 没有过期时间会永久死锁。SET...NX EX 一条命令原子完成加锁+设过期。
- **Q：** 解锁为什么必须用 Lua 脚本？
  **A：** 解锁需「判断锁是自己的 + 删除」两步。若分开执行，判断通过后锁恰好过期并被别人获取，DEL 会误删他人的锁。Lua 脚本在 Redis 中作为整体原子执行，避免这个竞态。
- **Q：** 有了 IO 多路复用和 6.0 多 IO 线程，分布式锁原子性还能保证吗？
  **A：** 能。IO 多路复用只是一次返回多个就绪 fd，主线程仍逐个串行处理；多 IO 线程只并行做读取/解析/写回，命令的实际执行始终在主 IO 线程串行完成，SET 和 EVAL 都是原子的，所以分布式锁原子性不受影响。

---

## 六、逐讲精要（缓存淘汰与持久化 15-20）

## 15 | 近似 LRU 算法的实现

**一句话核心**：Redis 出于省内存和保性能的考虑，没有用标准 LRU 链表，而是用 redisObject 的 lru 字段记录访问时间戳 + 随机采样近似淘汰。

**关键知识点**
- 内存淘汰由两个配置驱动：`maxmemory`（最大内存上限）和 `maxmemory-policy`（淘汰策略）。实际内存超过 maxmemory 时按策略淘汰。
- **8 种淘汰策略**：noeviction（默认，不淘汰、写入报错）、allkeys-lru、volatile-lru、allkeys-lfu、volatile-lfu、allkeys-random、volatile-random、volatile-ttl。`allkeys-*` 在所有 key 中筛选，`volatile-*` 只在设了过期时间的 key 中筛选。
- **为何不用标准 LRU**：标准 LRU 需维护一个覆盖所有数据的链表，既占额外内存，每次访问/插入又要移动链表节点，拖慢性能。Redis 因此实现"近似 LRU"。
- **近似 LRU 的省内存做法**：不用链表，只在每个值的 redisObject 里用 24 bit 的 `lru` 字段记录最近访问时间戳（秒级）。
- **全局 LRU 时钟**：`server.lruclock`，精度 1 秒（LRU_CLOCK_RESOLUTION=1000ms）；由 serverCron（默认 hz=10，每 100ms）周期性更新。所以间隔小于 1 秒的两次访问时间戳相同。
- **时间戳的初始化与更新**：createObject 创建键值对时初始化 lru；lookupKey 每次访问该 key 时更新 lru（调用 LRU_CLOCK）。
- **随机采样淘汰**：freeMemoryIfNeeded 判断内存超限后，每次随机采样 `maxmemory-samples`（默认 5）个 key，算空闲时间，放入固定大小（默认 16）的待淘汰候选池 EvictionPoolLRU，池按空闲时间排序，淘汰空闲时间最长的。不够则循环重复，直到释放足够内存。
- 计算已用内存时会扣除主从复制缓冲区；触发入口链路：processCommand → freeMemoryIfNeededAndSafe → freeMemoryIfNeeded。

**面试高频问答**
- **Q：** Redis 的近似 LRU 和标准 LRU 有什么区别？为什么这么设计？
  **A：** 标准 LRU 要维护全量数据链表，访问时移动节点，内存和性能开销大。Redis 近似 LRU 不用链表，只在 redisObject 的 lru 字段（24bit）记访问时间戳；淘汰时随机采样若干 key，从中挑空闲时间最长的淘汰。用少量精度损失换取内存和性能。
- **Q：** maxmemory-samples 有什么作用？调大调小的影响？
  **A：** 每次淘汰时随机采样的 key 数量，默认 5。调大越接近真实 LRU、结果更准，但 CPU 开销更大；调小更快但更不精确。
- **Q：** allkeys-lru 和 volatile-lru 区别？
  **A：** 前者从所有 key 中选淘汰对象；后者只从设置了过期时间的 key 中选。若 volatile-* 下没有可淘汰的带过期 key，会像 noeviction 一样对写入报错。
- **Q：** 有 8 种淘汰策略，分别是？
  **A：** noeviction、allkeys/volatile 各配 lru、lfu、random，外加 volatile-ttl（在带过期 key 中淘汰最快过期的），共 8 种。

---

## 16 | LFU 算法及其优势

**一句话核心**：LFU（4.0 引入）按访问频率淘汰，复用 lru 字段拆成"16bit 时钟 + 8bit 计数"，计数用概率对数递增、并按时间衰减，能更准地淘汰低频冷数据。

**关键知识点**
- **频率 ≠ 次数**：频率是单位时间内的访问次数。只记次数会让老数据（历史高频但现已不访问）迟迟不被淘汰。LFU 引入时间维度解决这个问题。
- **复用 lru 字段**：LRU 与 LFU 不会同时启用，所以 LFU 复用同一个 24bit lru 字段：**高 16 bit 存访问时间戳（分钟精度）**，**低 8 bit 存访问计数**（最大 255）。
- **初始化**：createObject 时，LFU 下 lru = (分钟时间戳 << 8) | LFU_INIT_VAL，初始计数 `LFU_INIT_VAL` 默认 5（避免新 key 一创建就被衰减到 0 而立即淘汰）。
- **访问更新（updateLFU 三步）**：① 先按距上次访问的时长**衰减**计数（LFUDecrAndReturn）；② 再按概率**递增**计数（LFULogIncr）；③ 更新时间戳。
- **时间衰减**：衰减量 = 距上次访问的分钟数 / `lfu-decay-time`（默认 1，即每过 1 分钟衰减 1）。解决"历史高频数据长期占用"问题。
- **概率对数递增**：计数越大越难增。阈值 p = 1/(baseval × `lfu-log-factor` + 1)，随机数 r<p 才加 1。baseval（当前计数-初始值）或 lfu-log-factor 越大，p 越小、越难增。这样用 8bit（最大 255）就能表示很大的访问频率区间。
- **淘汰流程与 LRU 相同**（同一套入口函数和候选池），区别在候选池 idle 值算法：LFU 用 `idle = 255 - 衰减后的访问次数`，访问次数越大 idle 越小、越不易被淘汰。

**面试高频问答**
- **Q：** LFU 和 LRU 的区别？各自适合什么场景？
  **A：** LRU 按"最近是否访问"（时间）淘汰，LFU 按"访问频率"淘汰。LRU 可能把偶尔被扫一次的冷数据当成热数据留下；LFU 因带频率衰减，更能识别并尽早淘汰真正的低频冷数据，适合有明显冷热区分、防偶发扫描污染缓存的场景。
- **Q：** 8bit 计数最大才 255，怎么表示高频访问？
  **A：** 不是每访问一次就加 1，而是概率递增：计数越大加 1 的概率越低（对数式增长）。由 lfu-log-factor 控制，用有限的 8bit 表达很大的频率范围。
- **Q：** 为什么访问 key 时要"先衰减再递增"计数？
  **A：** LFU 淘汰依据是频率而非累计次数。若只增不减，历史高频但现已不用的数据永远淘汰不掉。衰减按距上次访问的时长进行，长时间不访问频率就降下来，符合频率定义。
- **Q：** lfu-log-factor 和 lfu-decay-time 分别控制什么？
  **A：** lfu-log-factor 控制计数递增难度（越大越难增，频率区分越平缓）；lfu-decay-time 控制衰减速度（每多少分钟衰减 1，值越小衰减越快）。

---

## 17 | LazyFree（惰性删除）对缓存淘汰的影响

**一句话核心**：惰性删除（4.0+）把大 key 的内存释放交给后台线程异步执行，避免阻塞主线程；但它先把 key 从哈希表摘除、再评估代价决定异步与否，不会影响缓存淘汰的内存释放要求。

**关键知识点**
- **删除 = 两步子操作**：① 把 key 从哈希表摘除；② 释放 key/value 占用的内存。两步都做是**同步删除**；只做①、②交给后台线程是**异步删除（惰性删除）**。
- **4 个惰性删除配置**（默认都 no）：`lazyfree-lazy-eviction`（缓存淘汰）、`lazyfree-lazy-expire`（过期删除）、`lazyfree-lazy-server-del`（如 RENAME 隐式删旧 key）、`replica-lazy-flush`（从库全量同步前清空旧数据）。6.0 另加 `lazyfree-lazy-user-del`（让 DEL 也走异步）。
- **DEL vs UNLINK**：淘汰传播到 AOF/从库时，开启 lazy-eviction 用 UNLINK，否则用 DEL。DEL 同步释放，UNLINK "可能"异步释放。
- **异步不代表一定异步**：dbAsyncDelete 会先用 `lazyfreeGetFreeEffort` 评估释放代价（按集合元素个数），代价 ≤ 阈值 `LAZYFREE_THRESHOLD`(64) 时仍在主线程同步释放，只有大集合（>64 元素且非紧凑编码）才丢给后台线程（bioCreateBackgroundJob）。因为跨线程传递数据本身有开销，小对象没必要。
- 评估的是"释放工作量"而非内存大小：内存连续（如 String bigkey）代价低仍走主线程，所以**String 类型 bigkey 删除依然可能阻塞主线程** → 仍不建议存 bigkey。
- **不影响淘汰的内存释放**：异步删除时主线程会每删 16 个 key 检查一次当前内存是否已达标，满足即停止淘汰。后台线程也会及时消费任务队列释放内存，只是相比同步略有延迟。
- 同步释放最终经 dictFreeKey/dictFreeVal/zfree；value 释放走 decrRefCount（引用计数减到 1 才真正按类型释放）。异步释放的后台函数 lazyfreeFreeObjectFromBioThread 内部也调 decrRefCount。

**面试高频问答**
- **Q：** 什么是 LazyFree？解决什么问题？
  **A：** 惰性删除，4.0 引入。删除大 key 时释放大量内存会长时间阻塞主线程，LazyFree 把内存释放放到后台线程异步完成，避免阻塞。通过 lazyfree-lazy-* 系列配置在淘汰/过期/隐式删除/从库清库等场景开启。
- **Q：** 开启 LazyFree 后是不是所有删除都异步？
  **A：** 不是。Redis 会先评估释放代价（按集合元素数），只有大集合（>64 元素且非紧凑编码）才真正丢后台线程，小对象和内存连续的 String（即便是 bigkey）仍在主线程同步释放。
- **Q：** DEL 和 UNLINK 有什么区别？
  **A：** DEL 同步释放内存；UNLINK 先摘除 key、再评估决定是否异步释放（可能异步）。6.0 后可用 lazyfree-lazy-user-del 让 DEL 也具备 UNLINK 的效果。
- **Q：** 惰性删除后台线程释放内存期间，主线程还能处理请求吗？
  **A：** 能。主线程决定淘汰后先把 key 从全局哈希表摘除，客户端已查不到该 key，释放工作交后台线程后主线程立即继续处理请求，对客户端无影响。

---

## 18 | RDB 文件的生成与格式

**一句话核心**：RDB 是某一时刻内存数据的二进制快照，通过 fork 子进程 + 写时复制生成，文件小、加载快，但可能丢失最后一次快照后的数据。

**关键知识点**
- **RDB = 内存快照**：记录某一时刻全部键值对；二进制格式，体积比 AOF 小，写盘/加载快。
- **3 个创建入口函数**（rdb.c）：`rdbSave`（save 命令，前台）、`rdbSaveBackground`（bgsave，fork 子进程）、`rdbSaveToSlavesSockets`（主从无盘复制，直接把 RDB 字节流发给从节点）。最终都调 `rdbSaveRio` 写内容。
- **save vs bgsave**：save 在主线程执行、阻塞；bgsave fork 子进程生成、主线程继续处理请求（生产用 bgsave）。
- **fork + 写时复制（COW）**：bgsave fork 出子进程，父子共享内存页；父进程有写操作时才复制被修改的页，子进程仍基于 fork 那一刻的数据快照生成 RDB。fork 本身及大量写导致的 COW 会有开销。
- **其他触发时机**：flushall、正常 shutdown（prepareForShutdown）会触发 rdbSave；serverCron 会按 `save <seconds> <changes>` 配置触发 bgsave，主从复制落盘方式也触发 bgsave。
- **RDB 文件三部分**：文件头（魔数"REDIS"+ 版本号、redis 版本、创建时间、内存占用等属性 aux 字段）、数据部分（各 db 的所有键值对）、文件尾（EOF 结束标识 + 8 字节校验值 checksum，用于检测篡改/损坏）。
- **自包含格式**：用**操作码**标识内容（如 FA=aux 属性、FE=SELECTDB 库号、FB=RESIZEDB 哈希表大小、FC=毫秒过期时间、FF=EOF），用**类型码**标识 value 类型（如 0=String、13=Hash_ziplist）。每段按"类型/长度/实际数据"记录，便于解析。可整数编码的字符串用紧凑整数编码省空间。
- 键值对记录顺序：过期时间/LRU 空闲/LFU 频率 → value 类型 → key → value。
- 安全写入：先写临时文件再 rename 替换，掉电不会损坏旧 RDB（但需双倍磁盘空间）。

**面试高频问答**
- **Q：** save 和 bgsave 有什么区别？
  **A：** save 在主线程同步生成 RDB，期间阻塞所有请求；bgsave fork 一个子进程后台生成，主线程继续服务。生产环境用 bgsave。
- **Q：** bgsave 期间主线程还在写数据，如何保证快照一致？
  **A：** 靠 fork + 写时复制（COW）。fork 时父子共享内存页，子进程看到的是 fork 那一刻的数据；父进程后续修改会触发对应内存页复制，改的是副本，不影响子进程读取的原页，所以 RDB 是 fork 时刻的一致快照。
- **Q：** RDB 的优缺点？
  **A：** 优点：二进制紧凑、文件小、恢复快，适合备份和全量复制。缺点：快照有间隔，宕机会丢失最后一次快照后的数据；fork 和 COW 在大内存实例上有性能抖动。
- **Q：** RDB 文件由哪几部分组成？
  **A：** 文件头（魔数+版本+属性信息）、数据部分（各库全部键值对，用操作码/类型码标识）、文件尾（EOF 标识 + 校验值）。格式自包含，按类型-长度-数据记录便于解析。

---

## 19 | AOF 重写（上）：触发时机与影响

**一句话核心**：AOF 是"写后日志"，会越写越大；AOF 重写用 fork 子进程按当前数据的最新状态重新生成精简日志（记最终插入命令而非历史操作），避免阻塞主线程。

**关键知识点**
- **AOF = 写后日志**：先执行命令、再把写命令追加到 AOF 文件。与 RDB 记录数据本身不同，AOF 记录"命令 + 键值对"。
- **appendfsync 三种回盘策略**：`always`（每条命令都 fsync，最可靠最慢）、`everysec`（每秒 fsync，默认，折中）、`no`（交给操作系统决定何时刷盘，最快最不可靠）。
- **为何重写**：AOF 记录所有历史写操作，文件持续膨胀。重写只针对每个 key 的**最新状态**记录一条插入命令（如 String 记 SET、Set 记 SADD），丢弃历史冗余操作，使文件变小、恢复更快。
- **重写核心函数**：`rewriteAppendOnlyFileBackground`（fork 子进程）→ 子进程执行 `rewriteAppendOnlyFile` → `rewriteAppendOnlyFileRio`（遍历所有 db 的键值对生成命令）。
- **4 个触发时机**：① 执行 `bgrewriteaof` 命令；② config set appendonly yes 开启 AOF；③ AOF 重写被设为待调度（aof_rewrite_scheduled=1）后由 serverCron 择机执行；④ serverCron 周期检查：AOF 已开启 + 文件大小超基准比例 + 超绝对值下限。
- **两个自动重写阈值**：`auto-aof-rewrite-percentage`（相对基准的增长比例，默认 100%）、`auto-aof-rewrite-min-size`（绝对大小下限，默认 64MB）。
- **重写与 RDB 子进程互斥**：任一时机下若已有 RDB 子进程或 AOF 重写子进程在跑，重写都不立即执行（会置 aof_rewrite_scheduled=1 待调度）。因为两者都是 fork + 遍历全量数据 + 高磁盘 IO，同时跑会争抢 CPU/磁盘。
- **重写期间禁止 rehash**（updateDictResizePolicy）：rehash 会移动大量数据，加剧父进程内存修改 → 子进程更多写时复制，影响性能。
- 重写也用 fork 子进程避免阻塞主线程；但与 RDB 不同，重写期间主进程收到的新写操作需通过**管道**同步给子进程（见第 20 讲）。

**面试高频问答**
- **Q：** AOF 和 RDB 的区别？
  **A：** RDB 是二进制内存快照、记数据本身、文件小恢复快但可能丢数据；AOF 是写后日志、记命令、可靠性高（everysec 最多丢 1 秒）但文件大恢复慢。生产常用 AOF+RDB 混合。
- **Q：** appendfsync 三种策略怎么选？
  **A：** always 每命令刷盘最安全但性能差；no 由 OS 决定刷盘最快但宕机可能丢较多；everysec 每秒刷一次，是可靠性与性能的折中，最常用（最多丢 1 秒数据）。
- **Q：** 什么是 AOF 重写？为什么需要？怎么触发？
  **A：** AOF 记录所有历史写命令会不断变大，重写按每个 key 的最新状态只记一条插入命令来瘦身。触发：手动 bgrewriteaof、开启 AOF、待调度、以及文件大小同时超过 percentage 和 min-size 阈值时自动触发。
- **Q：** AOF 重写会阻塞主线程吗？
  **A：** 基本不会。重写由 fork 出的子进程完成，主线程继续处理请求。但 fork 瞬间、以及重写期间父进程写操作触发的写时复制会带来一定开销；且重写期间禁 rehash 以减少 COW。

---

## 20 | AOF 重写（下）：重写期间新写操作的记录

**一句话核心**：AOF 重写期间主进程新收到的写操作，既写入正常 AOF 缓冲，也存入 aof_rewrite_buf 缓冲块并通过管道发给重写子进程，最终追加进新 AOF 文件，保证重写日志不丢新操作。

**关键知识点**
- **核心问题**：重写子进程基于 fork 那一刻的数据生成新 AOF，但重写期间主进程仍在处理新写操作，这些新操作必须被补进新 AOF 文件，否则会丢数据。
- **AOF 重写缓冲区 aof_rewrite_buf_blocks**：主进程收到写操作时，feedAppendOnlyFile 除写正常 AOF 外，若有重写子进程在跑，还调 aofRewriteBufferAppend 把命令追加到 `aof_rewrite_buf_blocks` 列表。每个元素是 `aofrwblock`（含一块默认 10MB 的 buf），块满则再分配新块。
- **管道机制（pipe）**：父子进程通过内核缓冲区通信，单向、先进先出（一端写一端读）。双向通信需两个管道。fds[0] 读、fds[1] 写。
- **重写用 3 个管道**（aofCreatePipes 创建 6 个 fd）：① 操作命令传输管道（主→子，传新写命令）；② 子→父 ACK 管道；③ 父→子 ACK 管道。
- **命令传输流程**：主进程把新命令缓存进 aof_rewrite_buf_blocks → 注册写事件，回调 aofChildWriteDiffData 从缓冲块取数据经管道发给子进程 → 子进程 aofReadDiffFromParent 读取并累积到 `server.aof_child_diff` 字符串 → 最终 rewriteAppendOnlyFile 把 aof_child_diff 写入新 AOF 文件。
- **多次读取**：aofReadDiffFromParent 被 rewriteAppendOnlyFileRio、rewriteAppendOnlyFile、rdbSaveRio（混合持久化时）多次调用，尽可能多地补齐最新操作。
- **ACK 收尾**：子进程重写完成后向父进程发 "!"（让父停止发新命令），父进程回调 aofChildPipeReadable 回复 "!"，子进程 syncRead 读到确认后收尾，最后 rename 临时文件替换旧 AOF。类似一次握手确认。
- **混合持久化 aof-use-rdb-preamble**：4.0+ 支持，重写生成的 AOF 文件前半部分是 RDB 格式（快照，加载快），后半部分是重写期间新增的 AOF 命令。兼顾 RDB 恢复快和 AOF 丢数据少。（rdbSaveRio 参与重写即体现此机制）

**面试高频问答**
- **Q：** AOF 重写期间的新写操作会不会丢？如何保证？
  **A：** 不会。新写操作一方面写入正常的 AOF 缓冲/文件，另一方面被缓存进 aof_rewrite_buf_blocks 重写缓冲区，通过管道发给重写子进程，子进程读取后追加到新 AOF 文件末尾，保证新 AOF 包含重写期间的最新操作。
- **Q：** 重写期间父子进程怎么通信？用了几个管道？
  **A：** 用操作系统的管道（pipe，单向、FIFO）。共 3 个：1 个传新写命令（主→子），2 个 ACK 管道（子→父、父→子）用于重写结束时互相确认。
- **Q：** 什么是 AOF-RDB 混合持久化？有什么好处？
  **A：** 由 aof-use-rdb-preamble 开启（4.0+）。重写后的 AOF 文件头部是 RDB 二进制快照、尾部是重写期间新增的 AOF 命令。恢复时先快速加载 RDB 部分再重放少量命令，兼顾 RDB 加载快和 AOF 数据全的优点。
- **Q：** 为什么用重写缓冲区而不直接让子进程读父进程内存？
  **A：** fork 后子进程内存是 fork 时刻的快照，看不到之后主进程的新写操作。所以需要主进程把新命令单独缓存到 aof_rewrite_buf_blocks 并经管道传给子进程，才能把增量补进新 AOF 文件。

---

## 七、逐讲精要（主从·哨兵·集群 21-28）

## 21 | 主从复制：基于状态机的设计与实现

**一句话核心**：主从复制由从库发起，通过状态机（repl_state）驱动，历经初始化→建立连接→握手→复制类型判断四个阶段，最终以 PSYNC 命令决定执行全量复制还是增量复制。

**关键知识点**
- 复制有三种情况：全量复制（传 RDB 文件）、增量复制（传断连期间缺的命令）、长连接同步（主库把正常收到的写命令持续转发给从库）。
- 触发复制的三种方式：`replicaof masterip masterport` 命令、配置文件写 replicaof、启动参数 `--replicaof`。三者都只是让从库拿到主库 IP 和端口。
- 复制由从库主动发起，只有从库需要状态机；主库只是被动响应从库请求，无需维护状态机（否则还要处理灾备切换时的状态丢失、给每个从库 client 维护状态等问题）。
- 状态机变量 `repl_state`（存于 redisServer 结构）关键状态流转：NONE → CONNECT → CONNECTING → RECEIVE_PONG → 握手阶段(SEND/RECEIVE AUTH、PORT、IP、CAPA) → SEND_PSYNC → RECEIVE_PSYNC → TRANSFER → CONNECTED。
- 从库在周期任务 replicationCron（每 1000ms）中检查状态并连接主库；握手时相互 PING-PONG，从库上报自己的 IP、端口、对无盘复制和 PSYNC2 的支持。
- PSYNC 判断依据：从库发送主库 runid + 复制偏移量 offset。首次同步 offset 设为 -1，主库回 `+FULLRESYNC` 走全量；能续传则回 `+CONTINUE` 走增量；不支持返回错误。
- 全量复制流程：主库生成 RDB → 从库注册 readSyncBulkPayload 回调、状态置 TRANSFER → 从库清空自身数据、加载 RDB → 再接收主库缓冲的增量写命令。

**面试高频问答**
- **Q：** 全量复制和增量复制怎么选择？offset 和 runid 的作用？
  **A：** 从库发 PSYNC 带上主库 runid 和自己的复制偏移量 offset。首次或 runid 对不上（主库变了）→ 全量；断连后重连、且 offset 仍在主库的复制积压缓冲区（repl_backlog）范围内 → 增量，只补断连期间缺失的命令。offset 标记复制进度，runid 标识主库身份。
- **Q：** 主从复制为什么只给从库设计状态机？
  **A：** 复制发起方是从库，要经历连接、握手、请求数据多个阶段，用状态机能清晰管理状态流转、避免遗漏；主库只需被动响应，且主库可能故障切换导致状态丢失、还要给每个从库维护状态，收益低、代价高，所以不用。
- **Q：** 复制积压缓冲区（repl_backlog）是什么？
  **A：** 主库上一个固定大小的环形缓冲区，边同步给从库边把写命令也写进它。从库断连重连后，若 offset 还在缓冲区内即可增量复制；若已被覆盖则只能全量复制。缓冲区太小、主从断连久会退化为全量，是复制风暴的诱因之一。
- **Q：** 读写分离和复制风暴？
  **A：** 主写从读可分摊读压力，但主从有复制延迟，从库读到旧数据（最终一致）。复制风暴：主库同时给大量从库做全量复制会占满 CPU/带宽/内存，可用主从级联（从库再挂从库）缓解。

---

## 22 | 哨兵 Sentinel 的初始化

**一句话核心**：哨兵是运行在特殊模式下的 Redis server，与普通实例共用一套代码和 main 入口，通过启动参数区分，并从初始化起就用 Pub/Sub 与主节点、其他哨兵、客户端通信。

**关键知识点**
- 哨兵三大职责：监控（monitor）、自动故障转移（failover）、通知（notification）。
- 判断是否哨兵模式：`checkForSentinelMode` 检查命令是否为 `redis-sentinel`，或参数是否含 `--sentinel`，成立则 `server.sentinel_mode=1`。
- 哨兵专用初始化：`initSentinelConfig`（端口改为默认 26379、protected_mode 置 0 允许外部连接）+ `initSentinel`（替换命令表、初始化 sentinelState）。
- 命令表被清空后换成 sentinelcmds 数组；publish/info/role 等虽同名，但由哨兵专用函数（sentinelPublishCommand 等）实现。
- 核心结构：`sentinelState`（含哨兵 ID myid、当前纪元 current_epoch、监听的主节点哈希表 masters、TILT 模式标记等）。
- 通用实例结构 `sentinelRedisInstance` 靠 flags（SRI_MASTER/SRI_SLAVE/SRI_SENTINEL）区分它表示主节点、从节点还是其他哨兵；其中含 sentinels（监听同一主节点的其他哨兵）和 slaves（该主节点的从节点）两个哈希表。
- 启动时 `sentinelIsRunning`：确认配置文件可写（哨兵要把状态写回配置文件）、无 ID 则随机生成、向每个被监听主节点的 `+monitor` 频道发事件。

**面试高频问答**
- **Q：** 哨兵和普通 Redis 实例是两套程序吗？
  **A：** 不是，同一套源码、同一个 main 入口。启动时靠 `redis-sentinel` 或 `--sentinel` 判定为哨兵模式，然后用专用配置、专用命令表、专用事件机制运行。
- **Q：** 哨兵配置文件 sentinel.conf 在哪解析？
  **A：** main → loadServerConfig（读文件）→ loadServerConfigFromString（解析），其中对哨兵模式分支调用 sentinelHandleConfiguration 解析哨兵专属配置项。
- **Q：** 一个 sentinelRedisInstance 能表示几种角色？
  **A：** 三种——主节点、从节点、其他哨兵，靠 flags 字段区分，代码里通过判断 flags 走不同逻辑，复用同一结构体。

---

## 23 | 哨兵 Leader 选举与 Raft（上）

**一句话核心**：多个哨兵判断主库故障后，需按 Raft 协议选出唯一 Leader 来执行故障切换；哨兵平时对等（非 Leader/Follower），只在故障时才走 Raft 选举。

**关键知识点**
- 为什么要选举：多哨兵共同决策避免误判，但故障切换只能由一个哨兵执行，需就"谁是 Leader"达成分布式共识。
- Raft 三种角色：Leader、Follower、Candidate。正常时 Leader 向 Follower 发心跳；Follower 超时未收到心跳则转 Candidate 发起选举。
- Candidate 竞选：先给自己投一票 → 向其他节点发投票请求 → 启动计时器。收到 Leader 心跳则退回 Follower；收到超过半数确认票则成为 Leader。
- 每轮投票有轮次记录（term/纪元），Follower 每轮只能投一票；若无人过半（平票）则进入下一轮重新投票。
- 哨兵与标准 Raft 的区别：哨兵平时所有实例对等，没有 Leader/Follower，只有发现主库故障要切换时才按 Raft 的"投票选 Leader"规则临时选举。
- 时间事件入口 `sentinelTimer`（在 serverCron 中每 100ms 调用），对每个监听主节点调用 `sentinelHandleRedisInstance`。
- sentinelHandleRedisInstance 四步：①重连（sentinelReconnectInstance）②发送 PING/INFO/PUBLISH（sentinelSendPeriodicCommands）③判主观下线（sentinelCheckSubjectivelyDown）④仅对主节点：判客观下线+启动故障切换+Leader 选举。
- 主观下线 SDOWN 条件：距上次 PING 超过 `down-after-milliseconds`（默认 30s）仍无回复。判定后发 `+sdown` 事件、置 SRI_S_DOWN 标记。
- TILT 模式：哨兵两次时间事件间隔为负或过长（系统时间被改、资源繁忙等），进入 TILT 后只收集信息、不执行故障切换。
- sentinelTimer 末尾把 `server.hz` 加随机值，打散各哨兵执行频率，避免同时发起选举导致谁都拿不到多数票。

**面试高频问答**
- **Q：** 主观下线和客观下线区别？
  **A：** 主观下线（SDOWN）是单个哨兵在 down-after-milliseconds 内 PING 无响应就判定；客观下线（ODOWN）是多个哨兵都判主观下线、数量达到 quorum 阈值，才最终确认主库真的下线，可发起故障转移。前者防单点误判，后者才是切换依据。
- **Q：** 哨兵选举和 Raft 有何异同？
  **A：** 相同：都用任期/纪元、每轮一票、过半当选。不同：哨兵平时对等无 Leader/Follower 角色，也没有 Leader 心跳，只在主库故障需切换时临时按 Raft 规则选一个 Leader 执行切换。
- **Q：** 为什么哨兵要随机化 server.hz？
  **A：** 防止多个哨兵同频率同时发起选举，导致票被瓜分、每轮都选不出过半 Leader（类似活锁），随机打散能降低选举失败概率。

---

## 24 | 哨兵 Leader 选举与 Raft（下）

**一句话核心**：哨兵靠 `is-master-down-by-addr` 命令既询问客观下线又发起 Leader 投票；投票用纪元（epoch）保证一轮一票，得票超过半数且超过 quorum 者当选 Leader。

**关键知识点**
- 三个关键标记：主节点 flags 的 SRI_S_DOWN（当前哨兵判主观下线）、SRI_O_DOWN（判客观下线）；其他哨兵 flags 的 SRI_MASTER_DOWN（该哨兵也判主库主观下线）。
- 客观下线判断（sentinelCheckObjectivelyDown）：遍历 master->sentinels，quorum 计数=本哨兵(1)+其他标了 SRI_MASTER_DOWN 的哨兵；≥ master->quorum 阈值则 odown=1，发 `+odown` 事件、置 SRI_O_DOWN。
- SRI_MASTER_DOWN 从哪来：本哨兵通过 `sentinelAskMasterStateToOtherSentinels` 向其他哨兵发 `is-master-down-by-addr`，回复处理函数 sentinelReceiveIsMasterDownReply 若发现对方回 1（判了主观下线），就给对方置 SRI_MASTER_DOWN。
- `is-master-down-by-addr` 命令格式：`sentinel is-master-down-by-addr 主节点IP 端口 当前epoch 实例ID`。实例 ID 为 `*` 表示仅查询下线状态；为具体哨兵 ID 表示发起 Leader 投票。
- 该命令回复三部分：①对主节点主观下线的判断（0/1）②哨兵 Leader 的 ID ③Leader 所属纪元。所以一条命令兼具"询问下线"和"拉票"两个用途。
- 启动故障切换条件（sentinelStartFailoverIfNeeded）：主节点已标 SRI_O_DOWN、当前无正在进行的切换、距上次切换超过 failover-timeout 的 2 倍。满足则置 failover_state=WAIT_START、标 SRI_FAILOVER_IN_PROGRESS。
- 投票逻辑（sentinelVoteLeader）：比较请求哨兵纪元 req_epoch、本哨兵 current_epoch、master->leader_epoch。投票条件：master 记录的 leader 纪元 < 请求纪元，且请求纪元 ≥ 本哨兵纪元（保证一轮只投一次），否则返回已投的 Leader ID。
- 当选 Leader 两条件（sentinelGetLeader）：得票 > 半数（voters/2+1）且 ≥ quorum 阈值。
- 哨兵会判断从节点的主观下线，但只对主节点判客观下线和故障切换；被标下线的从节点在选新主时会被跳过。

**面试高频问答**
- **Q：** quorum 在客观下线和选举中分别是什么作用？
  **A：** 客观下线时，quorum 是判定主库客观下线所需的最少哨兵数（配置项）。选举 Leader 时，当选还要求得票既超过全部哨兵半数、又不少于 quorum。注意"判客观下线"和"选 Leader 过半"是两个不同门槛。
- **Q：** 一条命令怎么同时完成询问和拉票？
  **A：** `is-master-down-by-addr` 的实例 ID 参数为 `*` 时只询问下线状态；主库进入切换后该参数填成发起哨兵自己的 ID，接收方就会调用 sentinelVoteLeader 投票，回复里带上 Leader ID 和纪元。
- **Q：** 纪元（epoch）在选举里起什么作用？
  **A：** 相当于 Raft 的任期 term，作为投票轮次。通过比较纪元保证每个哨兵在同一轮只投一票，避免重复投票和脑裂。

---

## 25 | Pub/Sub 在主从故障切换中的作用

**一句话核心**：哨兵通过订阅主节点的 `__sentinel__:hello` 频道相互发现，并用 sentinelEvent 向各类频道发布事件，让客户端感知主观下线、客观下线、主库切换等进度。

**关键知识点**
- Pub/Sub 模型：发布者—频道—订阅者，多对多。适合哨兵与主从、其他哨兵、客户端之间的通信。
- 频道存储：`server.pubsub_channels` 是 keylistDictType 哈希表，key=频道名，value=订阅者列表（一个频道多个订阅者）。
- 发布 publishCommand → pubsubPublishMessage：查频道、遍历订阅者列表逐个投递。订阅 subscribeCommand → pubsubSubscribeChannel：把订阅者加入频道列表。
- 哨兵发布消息统一走 `sentinelEvent`（内部调 pubsubPublishMessage），用 C 可变参数（va_list/va_start/vsnprintf/va_end）生成不定长消息。
- 关键频道：`+monitor`（开始监控）、`+sdown`（主观下线）、`+odown`（客观下线）、`+switch-master`（主库切换完成，带旧主/新主的 name+IP+port）等。客户端订阅这些频道即可跟踪故障切换全过程。
- 哨兵互相发现：每个哨兵订阅所监听主节点的 `__sentinel__:hello` 频道（在 sentinelReconnectInstance 中通过 pc 连接 SUBSCRIBE）。
- 发布 hello：sentinelSendHello 向该频道 PUBLISH，内容含本哨兵 IP/端口/ID/当前纪元 + 主节点 name/IP/端口/config_epoch。
- 收到 hello：sentinelReceiveHelloMessages → sentinelProcessHelloMessage，若不认识该哨兵则 createSentinelRedisInstance 记录，从而监听同一主库的哨兵彼此发现。

**面试高频问答**
- **Q：** 哨兵之间是怎么互相发现的？
  **A：** 靠主节点的 `__sentinel__:hello` 频道。每个哨兵周期性向该频道发布自己的 IP/端口/ID/纪元和主库信息，同时订阅该频道，收到别的哨兵的 hello 就记录下来，从而无需人工配置即可发现监听同一主库的其他哨兵。
- **Q：** 客户端怎么知道主库已经切换、该连新主库？
  **A：** 客户端订阅哨兵的 `+switch-master` 频道，切换完成后哨兵发布该事件，携带旧主库和新主库的 IP/端口，客户端据此更新连接。也可订阅 +sdown/+odown 等了解中间进度。
- **Q：** 在哨兵实例上执行 publish 走哪个函数？
  **A：** 不是普通的 publishCommand。哨兵启动时替换了命令表，publish 由 sentinelPublishCommand 实现，主要处理 hello 频道消息。

---

## 26 | Ping-Pong 消息与 Gossip 协议（Redis Cluster）

**一句话核心**：Redis Cluster 去中心化，每个节点用 Gossip 协议通过随机挑节点收发 Ping/Pong 消息，把自身和已知节点信息（含 slots 分布）在集群内传播，最终各节点状态趋于一致。

**关键知识点**
- 集群状态维护两种思路：中心化（ZooKeeper/etcd 第三方）vs 去中心化（各节点自己维护、用 Gossip 传播）。Redis Cluster 选去中心化。
- 每个节点维护集群状态：各节点信息（名称/IP/端口）、运行状态（发 Ping 时间、收 Pong 时间）、数据分布（每个节点负责哪些 slot）。
- 四种消息类型：PING（发自身+已知节点信息）、PONG（回复 Ping，格式与 Ping 相同）、MEET（新节点加入集群）、FAIL（某节点故障）。
- 消息结构 `clusterMsg`：消息头含发送者 name/IP/通信端口 cport/负责的 myslots、消息类型、长度；消息体是联合体 clusterMsgData。Ping/Pong/Meet 用 clusterMsgDataGossip 数组，每个元素是一个已知节点的信息。
- Gossip 通信在 clusterCron（serverCron 每 100ms 调一次）中：每执行 10 次随机选 5 个节点，挑其中最久没发 Pong 的一个发 Ping，相当于每秒向一个随机节点发 Ping。
- clusterSendPing 三步：构建消息头（clusterBuildMessageHdr）、构建消息体（选 wanted 个节点写入，PFAIL 疑似故障节点放末尾）、clusterSendMessage 发送。wanted 默认约为集群节点数的 1/10（下限 3）。
- 收消息：节点连接上监听 clusterReadHandler，读到完整消息调 clusterProcessPacket；收到 Ping/Meet 用同一个 clusterSendPing 回 Pong；再用 clusterProcessGossipSection 处理消息体、更新本地节点信息和 slots 分布。

**面试高频问答**
- **Q：** Redis Cluster 为什么用 Gossip 而不用中心化组件？
  **A：** 去中心化无单点、无外部依赖，各节点平等维护状态、通过 Gossip 最终一致。代价是通信开销随节点数增大，这也是限制 Cluster 规模的关键因素。
- **Q：** Ping 和 Pong 消息内容一样吗？
  **A：** 一样，都由 clusterSendPing 生成，都携带发送者自身信息 + 已知的部分其他节点信息 + slots 分布。所以一来一回双方都能更新彼此掌握的集群视图。
- **Q：** Meet 和 Fail 消息什么时候用？
  **A：** Meet 是让一个新节点加入集群（对新节点发起握手）；Fail 是某节点被判定客观故障后广播，让全集群快速知道该节点下线。

---

## 27 | 从 MOVED、ASK 看集群节点如何处理命令

**一句话核心**：集群节点处理命令时先算 key 的 slot 定位所属节点，不在本地就重定向——slot 已归属其他节点回 MOVED（永久），slot 迁移中且 key 已迁走回 ASK（临时）。

**关键知识点**
- 集群节点命令处理仍是读取→解析→执行→返回四阶段，入口函数与单机相同，只在执行阶段（processCommand）针对集群模式加了重定向逻辑。
- processCommand 判断：`server.cluster_enabled` 开启、命令不来自本节点的主节点、命令带 key（或是 EXEC），才可能重定向。调 `getNodeByQuery` 查 key 所属节点，返回空或非本节点则调 `clusterRedirectClient` 重定向。
- getNodeByQuery 三步：①用 multiState 结构统一封装单条命令和 MULTI 事务的多条命令 ②对每个 key 用 `keyHashSlot`（CRC16 取模 16384）算 slot、查 slot 所属节点 ③按结果返回并设置 error_code。
- 多个 key 不在同一 slot → 报错 CLUSTER_REDIR_CROSS_SLOT。
- MOVED（CLUSTER_REDIR_MOVED）：key 的 slot 归属其他节点，返回 `MOVED slot IP:port`，客户端此后长期改向目标节点（永久重定向）。
- ASK（CLUSTER_REDIR_ASK）：slot 正在迁出且该 key 已不在本地（missing_keys>0），返回 `ASK slot 目标IP:port`，客户端仅本次去目标节点（临时重定向，需先发 ASKING）。
- 客观区别：MOVED 是槽已完成归属变更；ASK 是槽迁移进行中、单个 key 已迁走。集群客户端必须能识别并处理 MOVED/ASK 报错。

**面试高频问答**
- **Q：** MOVED 和 ASK 的区别？
  **A：** MOVED 表示 slot 已经永久归属另一个节点，客户端应更新本地槽映射、以后都访问新节点。ASK 表示 slot 正在迁移中、请求的这个 key 恰好已迁到目标节点，是临时重定向，客户端只这一次转向目标节点（且要先发 ASKING），不更新映射。
- **Q：** 集群怎么定位 key 属于哪个节点？
  **A：** 对 key 做 CRC16 再对 16384 取模得到 slot（keyHashSlot），再查 server.cluster->slots[slot] 得到负责该槽的节点。若不是本节点就重定向。
- **Q：** 一条命令的多个 key 跨 slot 会怎样？
  **A：** 返回 CROSSSLOT 错误。集群下多 key 操作要求所有 key 在同一 slot，可用 hash tag `{...}` 强制同槽。

---

## 28 | Redis Cluster 数据迁移会阻塞吗？

**一句话核心**：数据迁移按槽进行，用 clusterState 的 migrating/importing 数组标记状态；MIGRATE 用同步写 syncWrite/同步读 syncReadLine 传输，会阻塞源节点，故需控制单次迁移的 key 数量和大小。

**关键知识点**
- slot 归属记录：clusterNode.slots 是 CLUSTER_SLOTS/8 的 bitmap，每位表示是否负责该 slot（共 16384 个槽）。
- 集群级迁移状态 clusterState 四个结构：`migrating_slots_to[]`（本节点某槽正迁往谁）、`importing_slots_from[]`（正从谁迁入某槽）、`slots[]`（每槽归属哪个节点）、`slots_to_keys`（rax 字典树，快速查槽内有哪些 key）。
- 迁移五大步骤：①标记迁入迁出节点 ②获取待迁出 keys ③源节点实际迁移数据 ④目的节点处理数据 ⑤标记迁移结果。分别对应 CLUSTER SETSLOT / MIGRATE / RESTORE-ASKING 等命令。
- 标记：目的节点执行 `CLUSTER SETSLOT <slot> IMPORTING <源node>`，源节点执行 `CLUSTER SETSLOT <slot> MIGRATING <目的node>`（clusterCommand 处理）。
- 取 key：`CLUSTER GETKEYSINSLOT <slot> <count>`，经 countKeysInSlot/getKeysInSlot 返回槽内 key。
- 实际迁移（migrateCommand）：与目的节点建连 → 缓冲区填充 RESTORE-ASKING 命令+key+TTL+序列化 value（createDumpPayload 加 RDB 版本号和 CRC 校验和）→ syncWrite 按 64KB 分块发送 → syncReadLine 读回复。COPY/REPLACE 控制是否覆盖及是否删源 key。
- 目的节点（restoreCommand）：校验 replace/TTL → verifyDumpPayload 校验 RDB 版本和 CRC → rdbLoadObject 反序列化 → dbAdd 写入（有 REPLACE 先 dbDelete）→ 设置 TTL/LRU/LFU。
- 收尾：两端执行 `CLUSTER SETSLOT <slot> NODE <node>`，清 migrating/importing 标记、更新 slots 数组归属。
- 迁移中 key 的访问：命令处理时若槽在迁出且 key 已不在本地 → 返回 ASK 让客户端去目标节点；未迁走的仍在源节点正常处理。

**面试高频问答**
- **Q：** 数据迁移会阻塞吗？该注意什么？
  **A：** 会。MIGRATE 用 syncWrite 同步发送、syncReadLine 同步读回复，期间阻塞源节点正常处理请求。所以要控制单次迁移的 key 数量和 value 大小，避免一次迁移过多或过大的 key（尤其大 key）造成明显阻塞。
- **Q：** 迁移过程中客户端访问正在迁移的 key 会怎样？
  **A：** 若 key 还在源节点就正常处理；若 key 已迁到目标节点（本地 missing），源节点返回 ASK，客户端临时转向目标节点（发 ASKING 后执行）。迁移完成、执行 SETSLOT NODE 后，最终以 MOVED 稳定指向新节点。
- **Q：** 迁移的数据怎么保证正确性？
  **A：** value 用 createDumpPayload 序列化，结果尾部附 RDB 版本号和 CRC64 校验和；目的节点 restoreCommand 用 verifyDumpPayload 校验版本和校验和，不符则报错，确保传输完整且版本兼容。
- **Q：** 迁移状态用什么记录，槽和 key 怎么快速关联？
  **A：** clusterState 里 migrating_slots_to/importing_slots_from 记录迁出/迁入方向，slots 记录每槽归属；slots_to_keys 用 rax 字典树维护 slot→keys 映射，插入/删除 key 时同步更新，迁移时能快速取出某槽的所有 key。

---

## 八、逐讲精要（编程技巧模块 29-32）

## 29 | 如何正确实现循环缓冲区

**一句话核心**：循环缓冲区通过读写指针复用固定大小的空间循环读写，避免动态扩容；Redis 用它实现主从复制的复制积压缓冲区（repl_backlog），支撑断线后的部分重同步。

**关键知识点**
- 为什么用循环缓冲区：主从复制断连时需缓存主节点命令，数据发送后即可复用空间。相比"不够就动态分配"，循环缓冲区避免了分配开销和内存无限增长。
- 工作原理：一个写指针（写到末尾就回到头部覆盖旧数据）+ 一个或多个读指针（不同从节点各自的读位置，读到末尾也回头）。
- 两个全局偏移量概念：master_repl_offset（全局复制偏移量，主节点累积写入的命令总长度）、从节点全局读取位置（从节点累积读到的位置）。断线重连靠这两个值定位差量。
- Redis 数据结构（redisServer 中）：`repl_backlog`（char 数组）、`repl_backlog_size`（总长，对应配置 repl-backlog-size）、`repl_backlog_histlen`（当前累积数据长度，不超过总长）、`repl_backlog_idx`（写指针）、`repl_backlog_off`（缓冲区中最早数据首字节的全局偏移）。
- 多个从节点共享同一个 backlog（只在第一个从节点连接、且 backlog 为空时创建），避免每个从节点各开一块浪费内存。
- 实现两大难点：(1) 旧数据被覆盖后，如何知道"仍在缓冲区中最早的数据"的全局位置 → 用 repl_backlog_off 记录，公式 `off = master_repl_offset - histlen + 1`；(2) 读时如何把从节点的全局读取位置换算成缓冲区内位置 → 先算最早数据在缓冲区内的物理起点，再加上要跳过的长度 skip 并对总长取模。
- 写入可能跨越边界，需循环分段写（每轮最多写到缓冲区末尾）；读取同理，读到末尾要绕回头部继续。

**面试高频问答**
- **Q：** Redis 复制积压缓冲区（repl_backlog）是什么？为什么设计成循环/固定大小？
  **A：** 它是主节点缓存最近写命令的固定大小环形缓冲区，用于从节点断线重连后做部分重同步（partial resync）。固定大小循环复用避免动态扩容开销和内存膨胀；缺点是缓冲区太小、从节点断开太久时旧数据被覆盖，只能退化为全量同步。由 repl-backlog-size 配置。
- **Q：** 部分重同步是怎么定位断点、判断能否增量同步的？
  **A：** 从节点重连发 `PSYNC <runid> <offset>`；主节点比较该 offset 与 backlog 中最早保存数据的偏移 repl_backlog_off。若 offset 仍落在 backlog 覆盖范围内（skip = offset - repl_backlog_off 有效、且数据未被覆盖）就从对应位置读出差量增量发送，否则全量同步。
- **Q：** 一主多从时每个从节点各有一个 backlog 吗？
  **A：** 不是，所有从节点共享同一个 backlog，只是各自维护读取位置，节省内存。

## 30 | 如何在系统中实现延迟监控

**一句话核心**：Redis 用延迟监控框架（latency monitor）对可能导致变慢的几类"延迟事件"采样记录并给出优化建议，用 slowlog 列表记录超时的慢命令，二者结合定位延迟毛刺。

**关键知识点**
- 延迟监控框架 vs 慢日志的区别：延迟监控关注多类"延迟事件"（command、fast-command、aof-fsync-always、expire-cycle、fork 等系统级事件）；慢日志只记录执行时间超阈值的命令本身。
- 采样数据结构（latency.c/.h）：latencySample（采样时间 + 执行时长 ms）→ latencyTimeSeries（每类事件一个环形采样数组，默认 LATENCY_TS_LEN=160，满了覆盖旧样本）→ 全局哈希表 latency_events（事件名 → 采样序列）。
- 采样门槛：latencyAddSampleIfNeeded 宏只在事件执行时长 ≥ 配置项 latency-monitor-threshold 时才记录（该值为 0 表示关闭监控）。同一秒内的采样若更慢则覆盖前一个，避免同秒记录过多。
- 计时用 latencyStartMonitor / latencyEndMonitor 包裹事件，command 事件在 call 函数中采样。
- 排查命令：`LATENCY HISTORY <event>` 看历史采样，`LATENCY DOCTOR` 生成延迟分析报告和优化建议，`LATENCY LATEST` 看最新，`LATENCY RESET` 重置。
- LATENCY DOCTOR（createLatencyReport）不仅算均值/最大最小值，还结合配置给出针对性建议：启用 slowlog、调整 slowlog 阈值、检查 slowlog、避免 bigkey、减少磁盘竞争等。
- 慢日志（slowlog.c）：用 list 保存 slowlogEntry，记录命令及参数、执行时长（微秒）、时间戳、客户端名与地址。超过 slowlog-log-slower-than（微秒）才记录；列表超过 slowlog-max-len 就删表尾旧项。参数超过 SLOWLOG_ENTRY_MAX_ARGC（默认 32）只记部分并标注剩余个数，避免占内存。
- 慢日志只在命令执行的 call 函数中调用记录（slowlogPushEntryIfNeeded），且跳过 EXEC 命令（EXEC 只是触发事务，慢的是内部各命令，会各自记录）。

**面试高频问答**
- **Q：** 如何定位 Redis 的延迟毛刺 / 排查 Redis 变慢？
  **A：** 三条手段：(1) slowlog（SLOWLOG GET）查超阈值的具体慢命令，看是否操作了 bigkey、复杂度高的命令（如 KEYS、大范围 ZRANGE）；(2) latency monitor（LATENCY DOCTOR/HISTORY）查系统级延迟事件（fork、aof-fsync、过期删除等）并拿到优化建议；(3) INFO 命令看实时运行状态和资源使用。
- **Q：** slowlog 的两个关键配置是什么？记录的是命令排队时间还是执行时间？
  **A：** slowlog-log-slower-than（阈值，微秒，负数关闭、0 记录所有）、slowlog-max-len（列表最大长度）。记录的是命令实际执行耗时，不含网络和排队等待时间。
- **Q：** 慢命令和延迟事件采样一样吗？
  **A：** 不一样。慢日志只记录执行超时的命令，用列表存；延迟监控对多种可能致慢事件（含系统级）按类采样存到环形数组，并只在超过 latency-monitor-threshold 时采样。

## 31 | 从 Module 的实现学习动态扩展功能

**一句话核心**：Redis 4.0+ 通过 Module 机制以动态链接库（.so）形式扩展新命令和数据类型，模块用框架暴露的 RedisModule_* API 开发，新命令以"代理命令"方式注册进命令表。

**关键知识点**
- Module 是动态扩展手段：以 so 文件加载，可新增命令和数据类型，数据仍存在 Redis 内保证高性能访问。相关代码在 module.c 和 redismodule.h。
- 框架初始化（main 中 moduleInitModulesSystem）：创建待加载模块队列 loadmodule_queue、全局模块哈希表 modules，并 moduleRegisterCoreAPI 注册核心 API。
- API 命名转换：框架把 `RedisModule_Xxx` 的名字通过 REGISTER_API 宏映射到内部 `RM_Xxx` 实现函数，存入 moduleapi 哈希表（key=API 名，value=函数指针）。看到 RedisModule_ 前缀函数，就去找对应 RM_ 函数。
- 模块开发者必须实现两个函数：`RedisModule_OnLoad`（加载入口，框架用 dlsym 在 so 中查找并执行）和各命令的实现函数。OnLoad 里做两件事：RedisModule_Init 注册/初始化模块（并通过 GetApi 拉取框架 API 指针）、RedisModule_CreateCommand 注册新命令。
- 关键设计——代理命令：CreateCommand（RM_CreateCommand）并不把模块函数直接放进命令表，而是建一个 RedisModuleCommandProxy，其 redisCommand.proc 统一指向 RedisModuleCommandDispatcher，proxy.func 才记模块真正的实现函数。
- 命令执行链路：客户端命令 → processCommand 查命令表 → call 执行 proc（即 Dispatcher）→ RedisModuleCommandDispatcher 取出 proxy 并调用 proxy->func（模块真实函数）。多一层派发。
- 模块函数通过框架 API 复用 Redis 已有能力：如 RedisModule_ReplyWithString/ReplyWithLongLong 回复客户端、List Push/Pop、CreateString 等。

**面试高频问答**
- **Q：** Redis Module 机制是什么？能做什么？
  **A：** 4.0 引入的动态扩展机制，以 .so 动态库加载，可添加自定义命令和数据类型，数据存在 Redis 内。知名模块如 RedisJSON、RediSearch、RedisBloom、RedisTimeSeries。
- **Q：** 模块的新命令是怎么注册和执行的？为什么要"代理命令"？
  **A：** OnLoad 中调 RedisModule_CreateCommand 注册；框架在命令表里放的是代理命令，proc 统一指向 RedisModuleCommandDispatcher，由它再分发到模块真实实现函数。这样框架能在派发前后统一做上下文（RedisModuleCtx）准备、参数封装等处理，把模块函数和 Redis 内部命令调用协议解耦。
- **Q：** 开发一个模块最少要写什么？
  **A：** 必须有 RedisModule_OnLoad（用 RedisModule_Init 初始化模块 + RedisModule_CreateCommand 注册命令）和命令的实现函数（用 RedisModule_ReplyWith* 等 API 返回结果）。

## 32 | 如何在一个系统中实现单元测试

**一句话核心**：Redis 用 Tcl 语言编写单元测试框架，入口 test_helper.tcl 采用"测试 server + 多测试客户端 + 测试用例脚本"的三角色架构并发执行各功能模块的测试。

**关键知识点**
- 测试代码在源码 tests 子目录，用 Tcl（Tool Command Language，解释型、数据类型只有字符串）开发，无需编译。set 定义变量、$引用、`::`前缀是全局变量、proc 定义子函数、方括号执行命令取返回值。
- 运行方式：在 tests 目录层执行 `tclsh tests/test_helper.tcl`，不带参数则跑全局变量 ::all_tests 里预定义的所有用例。
- test_helper.tcl 三步：(1) source 引入 redis.tcl/server.tcl 等并定义全局变量（::all_tests、::host、::port、::single_tests 等）；(2) for 循环解析参数（--single 只跑指定用例、--client 表示当前是测试客户端置 ::client=1 等）；(3) 按 ::client 值决定跑 test_server_main 还是 test_client_main。
- 三个角色：
  - 测试 server（test_server_main）：socket -server 监听，启动默认 16 个（::numclients）测试客户端（exec 再次运行 test_helper.tcl 加 --client），通过 "run 用例名" 分发用例，周期性 test_server_cron 处理超时。
  - 测试客户端（test_client_main）：连上先发 "ready"，收到 "run 用例名" 调 execute_tests（source 引入并执行该用例 tcl 脚本），完成发 "done"。server 收 "ready"/"done" 用 signal_idle_client 继续派发未完成用例，实现并发。
  - 测试用例脚本（如 unit/type/string → string.tcl）：start_server 启动测试用 Redis 实例，用 r 函数发命令并断言返回值（如 `r set x foobar; r get x` 期望 foobar）。
- r 函数发命令的链路：r → srv → ::redis::redisHandle → ::redis::__dispatch__ → ::redis::__dispatch__raw__（按 RESP 协议封装命令发给测试实例）；函数名带 id 表示目标实例的 socket 描述符。
- 另有针对 SDS 的小型独立测试框架，在 sds.c 文件中（sdsTest 函数）。

**面试高频问答**
- **Q：** Redis 的测试框架用什么写的？为什么选它？
  **A：** 用 Tcl 语言写。Tcl 是解释型、上手快、擅长测试和运维脚本，无需编译，适合快速编写和执行覆盖各功能模块（数据类型、AOF/RDB、复制、集群等）的用例。
- **Q：** 这个单元测试框架的整体架构是怎样的？
  **A：** 三角色：测试 server 负责启动/调度多个测试客户端并分发用例；测试客户端向 server 报 ready、接收 "run 用例名" 后执行对应用例脚本、报 done；用例脚本用 start_server 起测试实例，用 r 函数发命令断言结果。server 与多客户端通过 socket 消息并发跑用例，提高测试效率。

