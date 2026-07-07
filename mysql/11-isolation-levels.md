# 11 · 事务隔离级别（Isolation Levels）

> 四种隔离级别 RU/RC/RR/Serializable 与脏读/不可重复读/幻读的对应关系，InnoDB 默认 RR。面试重要度：⭐⭐⭐ 高频（必考）。

## 📖 核心原理

隔离级别是"隔离性强弱"的档位，本质是**在并发正确性与性能之间做权衡**：隔离越强，并发副作用越少，但加锁越多、并发度越低。SQL 标准定义了四级，从弱到强：

**1. 读未提交 RU（Read Uncommitted）**：一个事务能读到另一个事务**未提交**的修改。最弱，几乎不用。会发生脏读、不可重复读、幻读。

**2. 读已提交 RC（Read Committed）**：只能读到**已提交**的数据，解决了脏读。但同一事务内两次读同一行，若中间别的事务提交了修改，两次结果不同——即**不可重复读**。Oracle、SQL Server 默认级别。

**3. 可重复读 RR（Repeatable Read）**：保证**同一事务内多次读同一行结果一致**，解决脏读和不可重复读。**InnoDB 的默认级别**。标准 SQL 中 RR 仍允许幻读，但 **InnoDB 的 RR 通过 MVCC + Next-Key Lock 基本解决了幻读**（见下文）。

**4. 串行化 Serializable**：最强，所有读都变成加锁的当前读（隐式给 `SELECT` 加共享锁），事务完全串行执行，杜绝一切并发问题，但并发度最低。

**三种并发读异常**（隔离级别要解决的问题）：

- **脏读（Dirty Read）**：读到别的事务**未提交**的数据，若对方回滚，读到的就是"幻影"脏数据。
- **不可重复读（Non-repeatable Read）**：同一事务内**两次读同一行**，因别的事务提交了 **UPDATE/DELETE**，结果不同。针对的是**已存在行的内容变化**。
- **幻读（Phantom Read）**：同一事务内**两次范围查询**，因别的事务提交了 **INSERT**，第二次多出（或少了）符合条件的行。针对的是**行数变化（新增/删除幻影行）**。

不可重复读和幻读容易混淆：**不可重复读关注"同一行值变了"，幻读关注"范围内行数变了（凭空多出行）"**。

**InnoDB RR 如何解决幻读。** 分两种读：

- **快照读**（普通 `SELECT`）：靠 **MVCC**，RR 下事务首次快照读时生成 ReadView 并固定，之后一直读这个一致性快照，别人插入的新行不在快照里，天然不出现幻读。
- **当前读**（`SELECT ... FOR UPDATE / LOCK IN SHARE MODE`、`UPDATE`、`DELETE`）：靠 **Next-Key Lock（临键锁 = 记录锁 + 间隙锁）**，锁住命中的记录及其前后间隙，阻止其他事务在范围内 INSERT，从而防止幻读。

所以"InnoDB RR 解决了幻读"要加限定：**快照读靠 MVCC、当前读靠 Next-Key Lock**，两者配合才完整。

**如何设置。**

```sql
-- 查看当前/全局隔离级别（8.0 用 transaction_isolation）
SELECT @@transaction_isolation, @@global.transaction_isolation;

-- 设置会话级
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- 设置全局级（对新连接生效）
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

## 🔄 原理图 / 流程剖析

**隔离级别 × 并发异常对照表**（✅=会发生，❌=已解决）：

| 隔离级别 | 脏读 Dirty Read | 不可重复读 Non-repeatable | 幻读 Phantom | 说明 |
|---|---|---|---|---|
| 读未提交 RU | ✅ 会 | ✅ 会 | ✅ 会 | 最弱，读到未提交数据 |
| 读已提交 RC | ❌ 解决 | ✅ 会 | ✅ 会 | Oracle 默认 |
| 可重复读 RR | ❌ 解决 | ❌ 解决 | ⚠️ 标准会/InnoDB 基本解决 | **InnoDB 默认** |
| 串行化 Serializable | ❌ 解决 | ❌ 解决 | ❌ 解决 | 加锁串行，最慢 |

隔离级别与副作用递进关系：

```mermaid
flowchart LR
    RU["RU 读未提交<br/>脏读+不可重复+幻读"] -->|加"只读已提交"| RC["RC 读已提交<br/>解决脏读"]
    RC -->|加"快照固定"| RR["RR 可重复读<br/>再解决不可重复读<br/>+幻读(InnoDB)"]
    RR -->|加"读也加锁"| S["Serializable<br/>解决全部"]
    style RR fill:#4a90d9,color:#fff
```

## 🔑 面试要点

- 四级从弱到强：**RU < RC < RR < Serializable**；隔离越强并发越差。
- 三异常：**脏读=读到未提交；不可重复读=同行值变了(UPDATE/DELETE)；幻读=范围内行数变了(INSERT)**。记住"不可重复读针对行内容、幻读针对行数量"。
- **RC 解决脏读，RR 再解决不可重复读，Serializable 解决全部**。
- **InnoDB 默认 RR**（多数数据库默认 RC）；这是历史原因：早期 binlog 只有 statement 格式，RC 下会导致主从数据不一致，RR 才安全。
- **InnoDB 的 RR 基本解决了幻读**：快照读靠 MVCC、当前读靠 Next-Key Lock，答题必须区分这两条路径。
- 设置用 `SET [SESSION|GLOBAL] TRANSACTION ISOLATION LEVEL ...`，8.0 变量名是 `transaction_isolation`（5.7 是 `tx_isolation`）。

## ❓ 高频面试题

**Q：脏读、不可重复读、幻读有什么区别？**
A：脏读是读到**其他事务未提交**的数据，对方一旦回滚就成了脏数据。不可重复读是同一事务内**两次读同一行**，因别的事务提交了 UPDATE/DELETE 导致结果不同，关注点是**已有行的内容变化**。幻读是同一事务内**两次范围查询**，因别的事务提交了 INSERT 导致第二次多出符合条件的"幻影行"，关注点是**结果集行数变化**。核心区别：不可重复读是"值变了"，幻读是"行数变了"。

**Q：InnoDB 的 RR 到底解决幻读了吗？怎么解决的？**
A：基本解决了，分两条路径。**快照读**（普通 SELECT）靠 MVCC：RR 下首次快照读生成 ReadView 后固定不变，后续都读这个一致性快照，别人新插入的行不在快照里，看不到幻影。**当前读**（FOR UPDATE、UPDATE、DELETE）靠 Next-Key Lock：锁住命中记录及前后间隙，别的事务无法在范围内 INSERT，从而防幻读。两者配合才完整——只说 MVCC 或只说锁都不全。仍有极端边界场景（如快照读后再当前读同范围）可能观察到幻读，但常规场景已解决。

**Q：为什么 MySQL 默认用 RR 而多数数据库用 RC？**
A：历史与主从复制原因。MySQL 早期主从复制依赖 **binlog 的 statement 格式**，在 RC 下，事务提交顺序与 binlog 记录顺序可能不一致，导致从库回放出的数据与主库不同（数据错乱）。RR 通过间隙锁保证了这种一致性，所以选为默认。虽然 8.0 有了 row 格式 binlog 缓解此问题，但默认级别沿用了 RR。

## ⚠️ 易错点 / 加分项

- **别混淆不可重复读和幻读**：一个是"行内容变了"、一个是"行数变了"。这是最常被追问的点。
- **RC 也用 MVCC，只是 ReadView 生成时机不同**：RC 每次快照读都重新生成 ReadView（所以能读到最新提交→不可重复读），RR 只在首次生成并固定（所以可重复读）。这是 12 篇的重点，别以为"MVCC 只有 RR 有"。
- **Serializable 的读是加锁的当前读**：它把普通 SELECT 隐式升级为加共享锁，靠加锁而非 MVCC 实现串行，性能差，一般不用。
- **加分点**：能指出"标准 SQL 的 RR 允许幻读，InnoDB 的 RR 是加强版"，并说明加强靠 Next-Key Lock，体现你分得清标准与实现。
- **加分点**：RC 相比 RR 少了间隙锁，**并发更高、死锁更少**，很多互联网大厂（如阿里规约倾向）在能接受不可重复读、且用 row 格式 binlog 的前提下，把生产库设为 RC 以提升并发，这是很实战的取舍。
