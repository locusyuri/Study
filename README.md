# Study — 开源源码学习仓库

个人源码学习项目：集中 clone 主流开源数据库、中间件与框架源码，并在 `wiki/` 下沉淀结构化学习笔记。所有子项目均为完整 git 仓库（独立 remote），可随时拉取更新或切换分支。

## 目录结构

| 目录 | 项目 | 版本 / 分支 | 上游仓库 |
|---|---|---|---|
| `Database/mysql-server` | MySQL Server | 8.4.11 LTS（分支 `8.4`） | github.com/mysql/mysql-server |
| `Database/redis` | Redis | 8.10.1（分支 `8.10`） | github.com/redis/redis |
| `Middleware/kafka` | Apache Kafka | 4.4（分支 `4.4`） | github.com/apache/kafka |
| `Middleware/rocketmq` | Apache RocketMQ | 5.3.3（分支 `master`） | github.com/apache/rocketmq |
| `SpringBoot/spring-boot` | Spring Boot | 3.2.x（分支 `3.2.x`） | github.com/spring-projects/spring-boot |
| `SpringBoot/spring-framework` | Spring Framework | 6.1.x（当前 6.1.22-SNAPSHOT） | github.com/spring-projects/spring-framework |
| `wiki/` | 学习笔记 | — | 自建，按主题建目录 |

## 项目定位

### Database —— 数据库内核

- **mysql-server**（C++）：MySQL 数据库服务器。重点学习 SQL 层（`sql/`）、InnoDB 存储引擎（`storage/innobase/`）、索引 B+ 树、事务 / MVCC / 日志恢复。
- **redis**（C）：Redis 内存数据库。重点学习事件驱动模型（ae 事件循环）、数据结构（dict / skiplist / ziplist）、持久化（RDB / AOF）。

### Middleware —— 消息中间件

- **kafka**（Java/Scala）：Apache Kafka 分布式消息系统。重点学习分区与副本机制、日志存储（log segment）、消费者组协调（GroupCoordinator）。
- **rocketmq**（Java）：Apache RocketMQ。重点学习消息存储（commitlog / consume queue）、高可用与主从同步、事务消息。

### SpringBoot —— Java 生态框架

- **spring-framework**（Java）：Spring 框架核心。重点学习 IoC 容器（BeanFactory / ApplicationContext）、AOP、事务抽象。
- **spring-boot**（Java）：Spring Boot。重点学习自动装配（AutoConfiguration）、起步依赖、内嵌容器。

## 学习笔记（wiki）

| 笔记 | 说明 |
|---|---|
| `wiki/MySQL/OVERVIEW.html` | MySQL 知识图谱总览：七大知识领域 → 源码落点，含 MVCC 专题与阅读路径 |
| `wiki/Redis/` | 待补充 |

笔记格式建议：主题目录 + 结构化 Markdown / HTML，内容尽量映射到具体源码文件。

## 约定

- 各子项目保持独立 git 仓库与上游 remote，便于 `git pull` 同步与分支切换。
- 学习笔记统一放在 `wiki/` 下，按主题建目录（如 `wiki/MySQL/`、`wiki/Redis/`）。
- `.gitignore` 已忽略 `*.png / *.jpg / *.jpeg` 等图片文件，笔记中的图片素材不入库。
- 根目录 `AGENTS.md` 用于记录 AI 协作约定，`README.md` 用于项目总览。

## 快速开始

```bash
# 查看某个项目的版本与状态
git -C Database/mysql-server log -1
git -C Database/redis branch --show-current

# 更新某个项目到上游最新
git -C Middleware/kafka pull
```

建议阅读顺序（以源码为线索）：先从 `wiki/MySQL/OVERVIEW.html` 建立整体知识图谱，再按"存储引擎 → 索引 → 事务 → 日志恢复 → 查询处理 → 复制"的依赖顺序深入对应源码目录。
