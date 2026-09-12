# Kafka 整体架构与核心概念 · 源码映射笔记

- 源码版本：Kafka 4.4.0（KRaft 架构，无 ZooKeeper）
- 源码位置：`Middleware/kafka`
- 语言构成：核心服务 Scala（core/）+ 新增 Java 化模块（server/、storage/、metadata/、raft/、group-coordinator/、transaction-coordinator/）+ 客户端 Java（clients/）

## 1. 整体架构：两个角色 + 四层

KRaft 时代集群只有两种进程角色，由 `process.roles` 决定，入口在 `core/src/main/scala/kafka/server/KafkaRaftServer.scala`：

| 角色 | 职责 | 入口类 |
|---|---|---|
| Broker | 处理生产/消费请求，管理分区日志、协调器 | `BrokerServer.scala`（core/.../kafka/server） |
| Controller | 集群元数据权威：建删主题、选主、ISR、Broker 注册 | `ControllerServer.scala` + `QuorumController.java`（metadata/.../controller） |
| Combined | 两者合并部署（单节点开发常用） | `KafkaRaftServer` 同时创建二者 |

两者共享 `SharedServer.scala`（Raft 客户端 + `__cluster_metadata` 元数据日志）。

四层结构：
1. 客户端层 `clients/`：Producer / Consumer / Admin
2. Broker 层 `core/ + server/`：网络与请求处理、分区读写与副本复制、协调器、元数据缓存
3. 存储层 `storage/`：UnifiedLog / LogSegment / 索引 / 压缩
4. 控制器层 `metadata/controller + raft/`：QuorumController + 控制管理器 + KRaft 共识

## 2. 核心概念 → 源码映射

| 概念 | 一句话 | 源码位置 |
|---|---|---|
| Topic / Partition / Log | 逻辑主题 → 物理分片 → 每分区一个追加日志 | `metadata/.../PartitionRegistration.java`（AR/ISR/Leader 元数据）；`storage/.../internals/log/UnifiedLog.java`；`LogManager.java` |
| Broker 注册与心跳 | Broker 启动向 Controller 注册并周期性心跳 | `metadata/.../controller/QuorumController.java`（registerBroker / processBrokerHeartbeat）；`ClusterControlManager.java`；`core/.../server/BrokerLifecycleManager`（server 模块） |
| Controller（KRaft 选举） | 元数据日志多数派复制，选出的 Leader 即活跃控制器 | `raft/.../raft/KafkaRaftClient.java`；`QuorumState.java`（LeaderState/FollowerState/CandidateState）；`RaftLog.java` |
| 元数据记录 | 一切元数据变更都是追加到 `__cluster_metadata` 的记录 | `metadata/.../metadata/` 生成类：TopicRecord、PartitionRecord、IsrRecord、BrokerRegistrationRecord 等 |
| 副本 / Leader / ISR | 每个分区多副本，一个 Leader 负责读写，ISR 为同步副本集合 | `core/.../kafka/cluster/Partition.scala`（makeLeader/makeFollower/inSyncReplicaIds/maybeExpandIsr/maybeShrinkIsr）；`ReplicationControlManager.java`（Controller 侧） |
| HW / LEO / LSO | 高水位（消费可见上限）、日志末端、日志起始 | `UnifiedLog.java`：highWatermark() / logEndOffset() / logStartOffset() |
| 副本拉取 | Follower 主动从 Leader 拉取数据追赶 | `core/.../kafka/server/ReplicaFetcherThread.scala`、`ReplicaFetcherManager.scala`、`AbstractFetcherThread.scala` |
| LogSegment 与索引 | 日志按段滚动，配偏移/时间稀疏索引 | `storage/.../internals/log/LogSegment.java`、`OffsetIndex.java`、`TimeIndex.java`、`AbstractIndex.java` |
| 日志恢复与清理 | 启动截断校验、按 retention 删除、compact 压缩 | `LogLoader.java`、`LogManager.java`、`LogCleaner.java` |
| 请求处理模型 | Reactor 网络 + 请求队列 + 处理线程池 + Purgatory 延时 | `core/.../kafka/network/SocketServer.scala`、`RequestChannel.scala`、`KafkaRequestHandler.scala`、`KafkaApis.scala`；`server-common/.../purgatory/DelayedOperation.java`、`server/.../purgatory/DelayedProduce.java`、`core/.../server/DelayedFetch.scala` |
| 消费组 | 组成员管理、分区分配、offset 提交 | `group-coordinator/.../group/GroupCoordinator.java`（接口）、`GroupMetadataManager.java`；客户端 `ConsumerCoordinator.java` |
| Offset 存储 | 提交的 offset 写入 `__consumer_offsets` 内部主题 | `metadata/.../controller/OffsetControlManager.java`；协调器端 `OffsetMetadataManager`（group-coordinator） |
| 事务 / 幂等 | producerId + sequence 幂等，跨分区事务 | `transaction-coordinator/.../transaction/TransactionStateManager.java`、`ProducerIdManager.java`、`TransactionMetadata.java`；客户端 `KafkaProducer.TransactionManager` |
| 元数据传播 | Controller 写日志 → 快照/增量 → 推送 Broker 缓存 | `metadata/.../image/MetadataImage.java`（不可变快照）、`image/publisher/MetadataPublisher.java`、`KRaftMetadataCachePublisher.java`；Broker 侧 `KRaftMetadataCache.java` |
| 内部主题 | `__consumer_offsets`、`__transaction_state`、`__cluster_metadata` | 见 `KafkaConfig`（offsets.topic.num.partitions 等）与对应模块 |
| 消息批次 | 生产者攒批，服务端按 RecordBatch 处理 | 客户端 `RecordAccumulator.java`、`Sender.java`；协议 `org.apache.kafka.common.record.MemoryRecords/RecordBatch` |

## 3. 一条消息的完整旅程

**生产**：`KafkaProducer.send()` → `RecordAccumulator.append()`（按分区攒批）→ `Sender.runOnce()` 批量发出 → Broker `SocketServer` → `KafkaApis.handleProduceRequest()` → `ReplicaManager.appendRecords()` → Leader 分区 `UnifiedLog.appendAsLeader()` 顺序写盘 → 若 `acks=all`，`DelayedProduce` 挂起等待 ISR 追齐 → 推进 HW → 返回。

**消费**：`KafkaConsumer.poll()` → `ConsumerCoordinator` 入组/分配 → `Fetcher` 发 `FetchRequest` → `KafkaApis.handleFetchRequest()` → `ReplicaManager.fetchMessages()` → `UnifiedLog.read()`（只能读到 HW 之前）→ 返回；offset 经 `OffsetCommit` 写入 `__consumer_offsets`。

**元数据**：Controller 收到请求 → 生成元数据记录追加到 `__cluster_metadata`（Raft 多数派复制）→ `MetadataPublisher` 推送增量 → Broker `KRaftMetadataCache` 更新 → 客户端通过 Metadata/心跳请求感知变更。

## 4. 建议阅读路线

1. 入口：`KafkaRaftServer.scala` → `BrokerServer.scala` / `ControllerServer.scala`（看组件装配）
2. 请求面：`KafkaApis.scala` → `ReplicaManager.scala`（appendRecords / fetchMessages）
3. 存储面：`UnifiedLog.java` → `LogSegment.java` / `OffsetIndex.java` → `LogManager.java` / `LogLoader.java`
4. 元数据面：`QuorumController.java` → `ReplicationControlManager.java` → `KafkaRaftClient.java` → `MetadataImage.java`
5. 副本与协调：`kafka/cluster/Partition.scala` → `ReplicaFetcherThread.scala` → `GroupCoordinator.java` → `TransactionStateManager.java`
6. 客户端：`RecordAccumulator.java` / `Sender.java` → `ConsumerCoordinator.java` / `Fetcher.java`
