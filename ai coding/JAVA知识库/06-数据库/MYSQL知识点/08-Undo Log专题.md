# Undo Log 专题

## 目录

- [1. 一句话理解 Undo Log](#1-一句话理解-undo-log)
- [2. Undo Log 解决什么问题](#2-undo-log-解决什么问题)
- [3. 一次数据修改的基本过程](#3-一次数据修改的基本过程)
- [4. Undo Log 记录什么](#4-undo-log-记录什么)
- [5. Undo Log 的类型](#5-undo-log-的类型)
- [6. 事务回滚](#6-事务回滚)
- [7. MVCC 与版本链](#7-mvcc-与版本链)
- [8. Read View 与 Undo 的分工](#8-read-view-与-undo-的分工)
- [9. 事务提交后 Undo 是否立即删除](#9-事务提交后-undo-是否立即删除)
- [10. Purge 与长事务](#10-purge-与长事务)
- [11. Undo 自身如何保证崩溃安全](#11-undo-自身如何保证崩溃安全)
- [12. 崩溃恢复中的 Undo](#12-崩溃恢复中的-undo)
- [13. Undo、Redo 与 Binlog](#13-undoredo-与-binlog)
- [14. 高频问题](#14-高频问题)
- [15. 核心结论](#15-核心结论)

## 1. 一句话理解 Undo Log

Undo Log 是 InnoDB 的撤销日志，保存撤销数据修改和构造历史版本所需要的信息，主要服务于：

1. 事务回滚。
2. MVCC 一致性读。

可以简单记忆：

```text
Undo：修改前是什么
Redo：数据页做了什么修改
```

## 2. Undo Log 解决什么问题

### 2.1 保证事务原子性

事务执行到一半失败或主动执行 `ROLLBACK` 时，InnoDB 可以利用 Undo 撤销已经完成的修改，使事务表现为“要么全部成功，要么全部失败”。

### 2.2 为 MVCC 提供历史版本

其他事务进行快照读时，如果当前记录版本对其不可见，可以沿着 Undo 版本链查找更早且可见的版本，从而减少读写之间的直接阻塞。

## 3. 一次数据修改的基本过程

执行：

```sql
UPDATE account
SET balance = balance - 100
WHERE id = 1;
```

可以简化为：

```text
1. 定位并锁定目标记录
2. 生成撤销本次修改所需的 Undo 信息
3. 修改 Buffer Pool 中的数据页
4. 生成 Redo，保护数据页和 Undo 页的修改
5. 事务执行 Commit 或 Rollback
6. 提交后的历史版本在无人使用时由 Purge 清理
```

这里的顺序是便于理解的逻辑流程，实际实现还涉及 Undo Page、Rollback Segment、锁和日志缓冲等内部结构。

## 4. Undo Log 记录什么

例如：

```sql
UPDATE account
SET balance = 900
WHERE id = 1;
```

修改前：

```text
balance = 1000
```

Undo 会保存足以将记录恢复到旧版本的信息。事务回滚时，可以将 `900` 恢复为 `1000`；快照读也可以利用这些信息构造历史版本。

需要注意：

- Undo 不是原始 SQL 的反向语句。
- Undo 不一定保存整行的完整副本。
- 它保存的是撤销修改或构造旧版本所需要的信息。

## 5. Undo Log 的类型

### 5.1 Insert Undo

插入记录时产生，主要用于事务回滚：

```text
INSERT 一行
   -> 事务回滚
   -> 删除新插入的记录
```

未提交的新记录对其他事务不可见，因此 Insert Undo 在事务结束后通常不需要继续为 MVCC 提供历史版本。

### 5.2 Update Undo

UPDATE 和 DELETE 等操作产生的 Undo，既可能用于回滚，也可能用于 MVCC 历史版本。

```text
balance：1000 -> 900

Undo：提供从 900 恢复或构造出 1000 所需的信息
```

Update Undo 在事务提交后可能仍然被其他事务的 Read View 使用，因此不能保证立即清理。

### 5.3 DELETE 的处理

InnoDB 删除记录通常先做删除标记，而不是立即物理删除：

```text
记录存在
   -> 标记为删除
   -> 事务提交
   -> 确认没有旧 Read View 需要
   -> Purge 执行物理清理
```

如果事务回滚，可以根据 Undo 撤销删除标记。

## 6. 事务回滚

执行：

```sql
BEGIN;

UPDATE account
SET balance = balance - 100
WHERE id = 1;

ROLLBACK;
```

可以理解为：

```text
balance：1000 -> 900
               -> 根据 Undo 逆向恢复
balance：900  -> 1000
```

一个事务包含多个修改时，回滚需要沿 Undo 记录按相反方向逐步撤销。大事务修改的数据越多，Undo 越多，回滚时间通常也越长，甚至可能接近或超过事务原本的执行时间。

Undo 也支持回滚到保存点：

```sql
SAVEPOINT sp1;

-- 执行若干修改

ROLLBACK TO sp1;
```

## 7. MVCC 与版本链

InnoDB 聚簇索引记录包含两个与 MVCC 密切相关的隐藏字段：

```text
DB_TRX_ID    最后修改该记录的事务 ID
DB_ROLL_PTR  指向相关 Undo 记录
```

一行记录被连续修改后，可以形成版本链：

```text
当前版本：balance = 700，trx_id = 30
       ↓ DB_ROLL_PTR
历史版本：balance = 800，trx_id = 20
       ↓
历史版本：balance = 900，trx_id = 10
```

快照读发现当前版本不可见时，会沿 `DB_ROLL_PTR` 查找更早的版本，直到找到对当前 Read View 可见的版本，或者版本链结束。

## 8. Read View 与 Undo 的分工

Read View 和 Undo Log 不是同一个概念：

```text
Read View：判断哪个版本可见
Undo Log：提供对应的历史版本
```

在 `READ COMMITTED` 和 `REPEATABLE READ` 下，普通快照读可能需要访问 Undo 版本链：

- `READ COMMITTED` 通常每次快照读创建新的 Read View。
- `REPEATABLE READ` 通常复用事务第一次快照读建立的 Read View。

当前读和写操作仍然需要锁，不能因为存在 Undo 和 MVCC 就认为所有操作都不加锁。

## 9. 事务提交后 Undo 是否立即删除

不一定。

事务提交后，生成 Undo 的事务已经不能再主动回滚，但其他事务可能仍需要通过该 Undo 读取旧版本：

```text
事务提交
   -> Update Undo 可能进入历史版本链
   -> 等待旧 Read View 不再需要
   -> Purge 线程清理
```

判断历史 Undo 能否清理的关键不是“生成它的事务是否已经提交”，而是：

> 是否还有活跃 Read View 可能访问这个历史版本。

## 10. Purge 与长事务

Purge 线程负责清理已经不再被任何活跃 Read View 需要的历史 Undo，并处理已经提交的删除标记记录。

如果存在长事务：

```text
长事务持有较老的 Read View
        ->
其他事务持续更新和删除数据
        ->
历史 Undo 不断积累
        ->
Purge 无法及时推进
```

可能造成：

- Undo 表空间增长。
- History List Length 增大。
- 记录版本链变长。
- 快照读遍历历史版本的成本增加。
- 删除记录不能及时物理清理。
- Purge 压力和磁盘 I/O 增加。

生产环境应避免无必要的长事务，尤其是开启事务后长时间空闲但一直不提交的连接。

## 11. Undo 自身如何保证崩溃安全

Undo Log 也存储在 InnoDB 管理的数据页中。写入或修改 Undo 页本身同样会产生 Redo Log：

```text
业务数据修改
   -> 生成 Undo，用于回滚和 MVCC
   -> Undo 页的修改由 Redo 保护
```

原因是数据库崩溃后可能仍然需要 Undo 回滚未提交事务，因此必须先保证 Undo 信息本身可以通过 Redo 恢复。

可以概括为：

> Undo 负责撤销业务修改，Redo 负责保证包括 Undo 页在内的数据页修改可以恢复。

## 12. 崩溃恢复中的 Undo

崩溃恢复可以从逻辑上理解为：

```text
第一阶段：Redo
恢复数据页和相关内部页面，可能包括未提交事务的修改
        ->
第二阶段：Undo
识别崩溃时未完成的事务，并撤销其修改
```

例如：

```text
T1：修改 A，已经提交
T2：修改 B，尚未提交
```

恢复时可能先通过 Redo 恢复 A 和 B，然后根据事务系统、Rollback Segment 和 Undo 状态判断：

```text
T1 已提交  -> 保留修改
T2 未提交  -> 使用 Undo 撤销修改
```

因此，Redo 回放出某个修改并不代表它最终一定保留；是否保留还取决于事务是否完成提交。

## 13. Undo、Redo 与 Binlog

| 日志 | 所属层 | 主要用途 | 核心关键词 |
|---|---|---|---|
| Undo Log | InnoDB | 回滚、MVCC | 旧版本、版本链 |
| Redo Log | InnoDB | 崩溃恢复、持久性 | WAL、重做页修改 |
| Binlog | MySQL Server | 复制、时间点恢复、订阅 | 逻辑变更 |

简单记忆：

```text
Undo：修改前是什么
Redo：数据页做了什么修改
Binlog：数据库发生了什么逻辑变更
```

三者不能相互替代：

- 没有 Undo，事务无法可靠回滚，MVCC 也缺少历史版本。
- 没有 Redo，已提交修改和 Undo 页在崩溃后可能无法恢复。
- 没有 Binlog，就缺少复制、时间点恢复和变更订阅所需的 Server 层日志。

## 14. 高频问题

### 14.1 Undo Log 只用于事务回滚吗？

不是。它还为 MVCC 快照读提供历史版本。

### 14.2 事务提交后 Undo 会立即删除吗？

不一定。只要还有活跃 Read View 可能需要该历史版本，Update Undo 就不能被 Purge 清理。

### 14.3 Undo 保存的是整行完整副本吗？

不一定。Undo 保存的是撤销修改或构造旧版本所需的信息，不能简单理解为每次都复制整行。

### 14.4 Undo 决定记录版本是否可见吗？

不是。Read View 负责可见性判断，Undo 负责提供历史版本。

### 14.5 为什么 Undo 还需要 Redo 保护？

数据库崩溃后仍可能需要 Undo 回滚未提交事务，所以 Undo 页本身也必须能够通过 Redo 恢复。

### 14.6 为什么大事务回滚很慢？

大事务产生的 Undo 多，回滚时需要按相反方向逐步撤销大量修改，并产生相应的数据页、索引页和日志开销。

### 14.7 长事务为什么会导致 Undo 堆积？

长事务持有较老的 Read View，Purge 无法确认旧版本已经无人使用，因此不能清理相关历史 Undo。

## 15. 核心结论

1. Undo Log 是 InnoDB 的撤销日志。
2. Undo 主要用于事务回滚和 MVCC 历史版本。
3. Undo 不是反向 SQL，也不一定保存完整行副本。
4. Insert Undo 主要用于回滚；Update Undo 还可能服务 MVCC。
5. `DB_TRX_ID` 记录修改事务，`DB_ROLL_PTR` 指向 Undo 版本。
6. Read View 决定版本是否可见，Undo 版本链提供历史数据。
7. 事务提交后，历史 Undo 不一定立即删除。
8. Purge 只能清理不再被任何活跃 Read View 需要的版本。
9. 长事务会阻碍 Purge，导致 Undo 和历史版本堆积。
10. Undo 页本身由 Redo Log 保护。
11. 崩溃恢复时先恢复页面，再使用 Undo 撤销未完成事务。
12. Undo、Redo、Binlog 职责不同，不能相互替代。

一句话总结：

> Undo Log 保存撤销修改和构造历史版本所需的信息，既保证事务能够回滚，也为 MVCC 提供旧版本；事务提交后是否清理，取决于是否还有活跃 Read View 需要这些版本。
