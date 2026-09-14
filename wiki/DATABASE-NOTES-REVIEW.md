# 数据库笔记：目录方案评审结论（供编写 Agent 使用）

> 用途：用户将编写一份内容丰富的数据库学习笔记（Part-Chapter-Section 三级目录）。
> 此前有一版 8-Part 目录设计，本文件为其评审结论与修订方向。后续编写该笔记时**直接遵循本文件**，不再沿用旧方案的缺陷结构。

## 一、旧方案必须修正的问题（A 级）

1. **Part 6 顺序存在依赖倒挂**（原：事务 → 日志 → MVCC → 锁）
   - `ch-transactions` 讲隔离级别 / 读现象，但其实现依赖锁（幻读 → Next-Key）与 MVCC（快照），内容被"向后引用"；
   - `ch-mvcc` 讲"快照读 vs 当前读"，"当前读"即加锁读，依赖 S/X 锁定义，而锁在最末。
   - **修正：事务 → 日志 → 锁 → MVCC**。逻辑：undo 为 MVCC 版本链供原料，锁为"当前读"供机制，MVCC 最后收尾成隔离级别实现的综合篇。

2. **"去重"不彻底，残留 3 处重复**
   - MVCC 拆两处：Part 4 讲 PostgreSQL 实现、Part 6 讲 MySQL 实现 → **合并到 Part 6 作对照小节**；
   - 页结构两处：Part 4 InnoDB 页 vs Part 5 B+ 树页结构 → **物理布局只在 Part 4 讲一次**，Part 5 只讲 B+ 树如何组织页（扇出/分裂/合并）；
   - 隔离级别两处：`ch-transactions` 定义 vs `ch-mvcc` 的 RC/RR 区别 → **transactions 只做定义与问题陈述**，行为差异与实现全部放 mvcc。

3. **粒度失衡，三章过载需拆分（每章一个主题）**
   - `ch-logging` 塞了 redo + undo + binlog + 2PC + 崩溃恢复（5 主题）；
   - `ch-engine-internals` 塞了引擎对比 + InnoDB 内部 + PG MVCC + 选型（4 主题）；
   - `ch-explain` 塞了慢查询 + EXPLAIN + 访问类型 + 连接算法 + 统计信息 + 反模式 + 改写 + 分区（8 主题）。

4. **内容缺口：备份与恢复、主从复制缺失**
   - binlog 讲了用途（复制），但没有主从复制 / GTID / 备份 / PITR 的章节，日志的用途未闭环；
   - **补充 `ch-replication-backup`**（主从复制 · GTID · 备份 · PITR）。

## 二、小问题（B 级，按需修正）

5. `ch-index-failure` 应在 `ch-explain` **之后**——索引失效的验证方法就是 EXPLAIN；
6. **2PC 概念必须区分**："单机 redo+binlog 对账"（类 XA 简化版）≠ "分布式事务 XA 2PC"，正文开头先说明；
7. `ch-analytics` 锚定为"**用 SQL / 窗口函数实现**留存 / 转化 / 同环比"，避免写成业务分析；与窗口函数章互引；
8. 权限统一收进 `ch-security`，`ch-views-functions` 只讲视图逻辑。

## 三、修订后骨架（Part 4 / 5 / 6 关键差异）

```
Part 4 存储引擎
  ch-engine-arch         通用架构 + MySQL 一条 SQL 的执行流程
  ch-engine-comparison   InnoDB / MyISAM / PG 对比与选型
  ch-innodb-internals    页物理布局 · 表空间 · 双写缓冲

Part 5 索引与查询优化
  ch-index-principles → ch-index-design → ch-explain → ch-index-failure（后移）→ ch-partition

Part 6 事务、日志与并发控制
  ch-transactions        ACID · 读现象 · 隔离级别（定义与问题陈述）
  ch-logging             redo · undo · 崩溃恢复
  ch-locking             S/X · 记录/间隙/临键锁 · 意向锁 · 死锁 · 悲观/乐观
  ch-mvcc                ReadView · 快照读 vs 当前读 · RC/RR 实现 · 长事务
  ch-replication-backup  主从复制 · GTID · 备份 · PITR
```

## 四、编写硬约束（沿用原方案的五条原则）

- **修错位**：存储引擎（架构/内部实现）前置到索引之前；索引建立在引擎之上；
- **去重复**：OLTP/OLAP 全站唯一定义（Part 1）；MVCC、页结构、隔离级别各只讲一次；
- **顺依赖**：日志 → 锁 → MVCC（undo 是版本链原料，当前读依赖锁）；索引依赖引擎；
- **清身份**：通用原理与 MySQL/PG 专有内容显式标注身份，默认以 MySQL 实现为主线；
- **控粒度**：每章一个主题，章节体量均衡，避免巨章。
