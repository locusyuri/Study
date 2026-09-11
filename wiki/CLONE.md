# 源码克隆命令说明

本项目下的 6 个开源项目源码均使用 **浅克隆（`--depth 1`）** 拉取，只保留目标分支的最新一次提交，大幅节省磁盘与拉取时间；适合以阅读学习为主、不需要历史记录的场景。如需完整历史，去掉 `--depth 1`，或克隆后执行 `git fetch --unshallow`。

## 命令总览

| 目标目录 | 项目 | 分支 | 克隆命令 |
|---|---|---|---|
| `SpringBoot/spring-boot` | Spring Boot | `3.2.x` | `git clone -b 3.2.x --depth 1 https://github.com/spring-projects/spring-boot.git` |
| `SpringBoot/spring-framework` | Spring Framework | `6.1.x` | `git clone -b 6.1.x --depth 1 https://github.com/spring-projects/spring-framework.git` |
| `Middleware/kafka` | Apache Kafka | `4.4` | `git clone -b 4.4 --depth 1 https://github.com/apache/kafka.git` |
| `Middleware/rocketmq` | Apache RocketMQ | `master` | `git clone -b master --depth 1 https://github.com/apache/rocketmq.git` |
| `Database/redis` | Redis | `8.10` | `git clone -b 8.10 --depth 1 https://github.com/redis/redis.git` |
| `Database/mysql-server` | MySQL Server | `8.4` | `git clone -b 8.4 --depth 1 https://github.com/mysql/mysql-server.git` |

## 分支说明

- **维护分支（推荐用于学习稳定版本）**：`3.2.x`（Spring Boot）、`6.1.x`（Spring Framework）、`4.4`（Kafka）、`8.10`（Redis）、`8.4`（MySQL LTS）
- **默认分支**：`master`（RocketMQ 以 master 跟踪最新发布，当前为 release 5.3.3）

## 使用方法

在 `C:\Repos\Study` 下执行，命令会自动创建对应项目目录：

```bash
# 示例：拉取 MySQL 8.4 LTS 源码
git clone -b 8.4 --depth 1 https://github.com/mysql/mysql-server.git Database/mysql-server
```

> 注意：直接按仓库名执行会克隆到当前目录下的同名目录（如 `mysql-server`），需自行移动或显式指定目标路径，使其落在 `Database/`、`Middleware/`、`SpringBoot/` 对应目录下。

## 常用维护命令

```bash
# 拉取目标分支最新代码（浅克隆下仍有效）
git -C Database/mysql-server pull

# 由浅克隆转为完整历史
git -C Database/mysql-server fetch --unshallow

# 查看当前分支与最近提交
git -C Database/mysql-server branch --show-current
git -C Database/mysql-server log -1 --oneline
```
