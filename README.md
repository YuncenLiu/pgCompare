

<div>
  <h1 style="font-size: 70px;text-align: center">pgCompare</h1>
  <h2 style="text-align: center">数据一致性对比工具</h2>
</div>
<hr>

[![License](https://img.shields.io/github/license/CrunchyData/postgres-operator)](LICENSE.md)

# 数据一致性对比变得简单

**pgCompare** 是一个基于 Java 的工具，用于在数据库复制或迁移后验证数据的一致性。它适用于以下场景：

- **从 Oracle/DB2/MariaDB/MySQL/MSSQL 迁移到 Postgres：** 在迁移过程中或迁移后进行数据对比。
- **相同或不同数据库平台之间的逻辑复制：** 跨平台验证数据一致性，同时最小化数据库开销。
- **主-主复制配置：** 定期检查数据一致性以降低风险。

pgCompare 使用哈希算法来高效比较表数据。主键和其余列的哈希值会被存储在一个仓库中，减少存储和网络需求。比较操作是并行处理的，从而提高了性能。

这个开源项目由 **Crunchy Data** 维护，并采用 **Apache 2.0 许可证** 发布，以便更广泛的使用、测试和反馈。

# 功能特性

- 支持 Oracle、PostgreSQL、DB2、MariaDB、MySQL 和 MSSQL。
- 使用哈希进行高效的并行数据比较。
- 支持批量处理以优化性能。
- 将多个对比项目的配置集中存储在一个仓库中。

# 安装指南

## 前提条件

在开始构建和安装之前，请确保满足以下前提条件：

1. **Java 21 或更高版本**
2. **Maven 3.9 或更高版本**
3. **PostgreSQL 15 或更高版本**（作为仓库数据库）
4. 支持的 JDBC 驱动程序（目前支持 DB2、PostgreSQL、MySQL、MSSQL 和 Oracle）。
5. 直接连接 PostgreSQL（例如不支持通过 pgBouncer）。

## 局限性

- 日期/时间戳仅比较到秒级别（格式：DDMMYYYYHH24MISS）。
- 不支持的数据类型：blob、long、longraw、bytea。
- 布尔类型在跨平台比较时存在限制。
- 低精度类型（float、real）不能与高精度类型（double）直接比较。
- 不同数据库对 float 的转换可能不同。如果在 float 数据类型的比较中出现问题，可以使用 float-cast 选项在 char 和 notation（科学计数法）之间切换。

# 快速入门

## 1. Fork 仓库

## 2. 克隆并构建项目

```shell
git clone --depth 1 git@github.com:<your-github-username>/pgCompare.git
cd pgCompare
mvn clean install
```


## 3. 配置属性文件

将 [pgcompare.properties.sample](file:///Users/xiang/xiang/study/Project/openPro/pgCompare/pgcompare.properties.sample) 复制为 [pgcompare.properties](file:///Users/xiang/xiang/study/Project/openPro/pgCompare/pgcompare.properties)，并更新其中的仓库、源数据库和目标数据库的连接参数。

默认情况下，应用程序会在执行目录中查找属性文件。您可以通过设置环境变量 `PGCOMPARE_CONFIG` 来指定自定义的属性文件路径。

最简配置需要在属性文件中设置 `repo-xxxxx` 类参数（或者通过环境变量）。除了属性文件和环境变量外，还可以将配置信息以 JSON 格式（{"parameter": "value"}）存储在 `dc_project` 表的 `project_config` 字段中。某些系统参数（如 log-destination）只能通过属性文件或环境变量指定。

## 4. 初始化仓库数据库

运行以下命令或脚本来初始化 PostgreSQL 仓库：

```shell
java -jar pgcompare.jar --init
```


## 5. 自动发现表结构

自动发现并映射指定 schema 中的表：

```shell
java -jar pgcompare.jar --discovery
```


# 使用方法

## 定义表映射

1. **自动发现**

   自动发现并映射指定 schema 中的表：

   ```shell
   java -jar pgcompare.jar --discover
   ```


2. **手动注册**

   手动向仓库中的 `dc_table` 和 `dc_table_map` 表插入映射关系。

## 执行数据对比

```shell
java -jar pgcompare.jar --batch 0
```


批次 0 表示处理所有数据。您可以使用 `PGCOMPARE-BATCH` 环境变量或通过 `--batch` 参数指定特定的批次号。

## 重新检查差异数据

重新校验标记有问题的行：

```shell
java -jar pgcompare.jar --batch 0 --check
```


# 升级说明

## 版本 0.3.0 新增功能

- 增加了对 DB2 的支持。
- 支持区分大小写的表名/列名。
- 新增了项目配置功能，便于管理多个对比任务。

**注意：** 升级到 0.3.0 时需删除并重新创建仓库数据库。

# 高级配置

## 属性配置方式

属性可通过属性文件、环境变量或 `dc_project` 表进行配置。环境变量优先于属性文件，并且必须以 `PGCOMPARE_` 为前缀。

示例：
- 属性文件：`batch-fetch-size=2000`
- 环境变量：`PGCOMPARE_BATCH_FETCH_SIZE=2000`

## 性能调优建议

- **批次大小：** 调整 `batch-fetch-size` 和 `batch-commit-size` 以提高内存效率。
- **线程数量：** 使用 `loader-threads`（默认值：4）进行并行处理。
- **观察者节流：** 启用 `observer-throttle=true` 以防止临时表过载。
- **Java 堆内存：** 对于大数据集，可能需要增加 Java 堆内存。可在执行时使用 `-Xms` 和 `-Xmx` 参数设置（例如：`java -Xms512m -Xmx2g -jar pgcompare.jar`）。

## 仓库数据库推荐配置

- 最小要求：2 vCPUs，8 GB RAM。
- PostgreSQL 推荐配置：
    - shared_buffers=2048MB
    - work_mem=256MB
    - max_parallel_workers=16

## 多项目支持

pgCompare 支持多个对比项目，每个项目通过 [pid](file:///Users/xiang/xiang/study/Project/openPro/pgCompare/src/main/java/com/crunchydata/pgCompare.java#L53-L53)（项目 ID）进行区分。这样可以在同一个仓库中管理不同的对比任务。如果没有指定项目，默认使用 `pid = 1`。

# 查看结果

## 上次运行的摘要信息

```sql
WITH mr AS (SELECT max(rid) rid FROM dc_result)
SELECT compare_start, table_name, status, equal_cnt+not_equal_cnt+missing_source_cnt+missing_target_cnt  AS total_cnt,
       equal_cnt, not_equal_cnt, missing_source_cnt + missing_target_cnt AS missing_cnt
FROM dc_result r
         JOIN mr ON (mr.rid = r.rid)
ORDER BY table_name;
```


## 不一致的记录

```sql
SELECT COALESCE(s.table_name, t.table_name) AS table_name,
       CASE
           WHEN s.compare_result = 'n' THEN 'out-of-sync'
           WHEN s.compare_result = 'm' THEN 'missing target'
           WHEN t.compare_result = 'm' THEN 'missing source'
           END AS compare_result,
       COALESCE(s.pk, t.pk) AS primary_key
FROM dc_source s
         FULL OUTER JOIN dc_target t ON s.pk = t.pk and s.tid=t.tid;
```


# 参考文档

## 属性分类

属性分为四类：系统、仓库、源数据库和目标数据库。详细描述如下：

### 系统属性

- batch-fetch-size：控制从源或目标数据库获取数据的批大小。
- batch-commit-size：控制提交到 `dc_source/dc_target` 表的记录数。
- batch-progress-report-size：用于报告进度的记录数。
- database-source：是否在源/目标数据库上排序（默认 true）。
- float-cast：定义浮点数如何参与哈希计算（notation|char），默认 char。
- loader-threads：加载数据的线程数，默认 4。
- log-level：日志级别。
- log-destination：日志输出位置，默认 stdout。
- message-queue-size：消息队列大小，默认 100。
- number-cast：数字转换方式（notation|standard），默认 notation。
- observer-throttle：启用观察者节流机制。
- observer-throttle-size：加载多少条数据后暂停。
- observer-vacuum：是否在检查点执行 vacuum。

### 仓库属性

- repo-dbname：仓库数据库名称。
- repo-host：仓库数据库主机。
- repo-password：仓库数据库密码。
- repo-port：仓库数据库端口。
- repo-schema：仓库数据库模式。
- repo-sslmode：SSL 模式（disable|prefer|require）。
- repo-user：仓库数据库用户名。

### 源数据库属性

- source-database-hash：是否在源数据库计算哈希。
- source-dbname：源数据库名称。
- source-host：源数据库主机。
- source-password：源数据库密码。
- source-port：源数据库端口。
- source-schema：源数据库 schema 名称。
- source-sslmode：SSL 模式。
- source-type：源数据库类型（oracle, postgres）。
- source-user：源数据库用户名。

### 目标数据库属性

- target-database-hash：是否在目标数据库计算哈希。
- target-dbname：目标数据库名称。
- target-host：目标数据库主机。
- target-password：目标数据库密码。
- target-port：目标数据库端口。
- target-schema：目标数据库 schema 名称。
- target-sslmode：SSL 模式。
- target-type：目标数据库类型（oracle, postgres）。
- target-user：目标数据库用户名。

## 属性优先级顺序

属性可以通过默认值、属性文件、环境变量或 `dc_project` 表配置。优先级顺序如下：

1. 默认值
2. 属性文件
3. 环境变量
4. `dc_project` 表中的配置

# 许可协议

**pgCompare** 遵循 [Apache 2.0 许可协议](LICENSE.md)。