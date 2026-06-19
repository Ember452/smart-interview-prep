# 大数据平台知识库

> 面试高频问题与答案要点。覆盖计算引擎/存储/消息队列/数据治理方向。

---

## 1. 计算引擎

### 1.1 Spark
- **Spark vs MapReduce** — Spark 基于内存计算（RDD/DataFrame 中间结果在内存中复用，避免多次 HDFS 读写）；MR 每步都落磁盘（shuffle 数据写磁盘再读）。Spark 适合迭代计算/机器学习/流处理；MR 仅适合批处理但更稳定/资源占用可控
- **shuffle 过程与调优** — Stage 间数据重分区（map 端写磁盘+索引文件 → reduce 端拉取+排序+聚合）。优化：减少 shuffle 数据量（filter 前置/spark.sql.shuffle.partitions 合理设值不盲目增加/提高 map 端 combine 预聚合减少数据）、提高磁盘并行（调多 executor core/local dir 分散盘 IO）
- **数据倾斜处理** — 某 key 数据量远大于其他→该 Task OOM/跑慢。方案：加随机前缀打散（两阶段聚合：第一轮 key+随机数分散聚合 → 第二轮去掉随机数再总聚）；广播小表替代 join（map 端本机 hash join，不发 shuffle）；过滤超热 key 走单独处理链路

### 1.2 Flink
- **Checkpoint 与状态管理** — Checkpoint 异步分布式快照（Chandy-Lamport 算法/Barrier 对齐），发生故障时从最近成功的 Checkpoint 恢复到状态+数据消费位置。Exactly-Once 语义由两端保证：source 端（Kafka offset 随 checkpoint 一起保存）→ sink 端（两阶段提交：预写事务+commit 事务）→ end-to-end 无重复
- **状态后端选择** — HashMapStateBackend（堆内存中/快但内存有限/GC 压力大/生产已废弃）；RocksDBStateBackend（磁盘上/可存海量状态/快照增量且异步/生产主流）。EmbeddedRocksDB 需留意磁盘 IO 和 compaction 影响 checkpoint 速度
- **Watermark 与乱序处理** — Watermark 表示事件时间进度（比最新 event time 滞后 乱序容忍），窗口触发时机=Watermark>=窗口结束时间。Watermark 设置太小→结果早出但可能不完整；太大→结果延迟大+状态堆积。Side Output 捕获迟到数据另处理

---

## 2. 存储

### 2.1 数仓 vs 数据湖
- **数据湖核心特征** — 原始格式直接存储（Parquet/ORC/Avro 开放格式而非专有格式 Hive）、Schema-on-Read（读时推断 schema 不强制入库建表）、存算分离（与计算引擎解耦/多引擎可读同一份湖数据，降低成本）。架构：湖仓一体（Lakehouse，Delta Lake/Iceberg/Hudi 在湖上增加事务/兼容 SQL/时间旅行/数据版本）
- **Parquet vs ORC vs Avro** — Parquet 列存（按列存储，列聚合查询/同列数据压缩比高，适合分析场景）；ORC 列存+stripe index+布隆过滤器（对点查/范围查比 Parquet 更有优势，Hive 生态专有）；Avro 行存（按行存储，适合逐行读写/流处理/Kafka 消息体/schema evolution 能力强）
- **小文件治理** — 小文件过多导致 NameNode 压力/每次查询打开文件数过多。方案：合并小文件（Hive compaction/Spark coalesce 合并分区内小文件）；设置合理分区粒度（避免过度分区如按秒级产生海量小文件）；流处理 mini batch 攒够一批再写

### 2.2 分区策略
- **分区键选择原则** — 查询高频过滤字段 + 分区数合理（分区数太大=元数据压力/分区数太小=扫描数据量大/效果差）。不适合做分区的：高基数列（user_id 百万分区）/频繁更新列。静态分区（INSERT 时明确指定分区值，快）/动态分区（Hive/Spark 根据字段值自动创建分区，慢且有风险/需有限制值）
- **分区生命周期管理** — 自动清理过期分区减轻存储压力和查询扫描范围；冷热数据分层（近 30 天 SS/HDD → 30-90 天低频存储 → 90+ 天归档/压缩/或直接删除）

---

## 3. 消息队列（Kafka）

- **Kafka 高吞吐设计** — 顺序 IO（追加写日志、无需随机修改，批量发送+压缩+ page cache 读）+ 零拷贝（sendfile 系统调用，数据不经过用户态直接磁盘→socket，避免两次拷贝）+ 分区并行（不同分区独立并发读写）
- **ISR 与数据可靠性** — ISR（In-Sync Replicas）为已与 Leader 同步的副本集合。acks=all 时生产者等待 ISR 中所有副本确认才返回成功（最高可靠性）。min.insync.replicas=2（至少 2 个 ISR 副本，低于该值分区不可写，保障一致性+可用性平衡）
- **Unclean Leader Election** — 非 ISR 副本被选举当 Leader（丢失已提交的数据），性能/可用性优先开 unclean=true；数据一致性优先设置 unclean=false（严格禁止非 ISR 当选，分区不可用直至 ISR 恢复）
- **Rebalance 触发 & 优化** — 消费者数量变化/消费者心跳超时（session.timeout.ms 默认 45s →新版本 max.poll.interval.ms 为两次 poll 之间的超时）触发 rebalance。解决：减少 rebalance 频率（调大两超时值）/静态组成员身份（group.instance.id 固定，滚动升级不触发 rebalance）/cooperative rebalance（增量重分配，不暂停整个消费组）

---

## 4. 实时数仓与 OLAP

- **实时数仓架构演进** — Lambda（批流双链路，架构复杂但稳定，批流结果需 Merge）→ Kappa（纯流式，简化架构但回溯难）→ 湖仓一体/实时数仓（Flink + StarRocks/Doris/ClickHouse 实时写入，统一存储与查询）
- **OLAP 引擎对比** — ClickHouse（列存 + MergeTree 引擎，单表查询极快，JOIN 弱，适合日志/指标分析）；Apache Doris/StarRocks（MPP 架构，多表 JOIN 能力强，实时更新，适合实时报表）；Presto/Trino（计算下推+无状态 Worker，适合联邦查询跨数据源，不存数据）
- **CDC（变更数据捕获）** — 原理：监听 DB binlog/WAL 捕获 INSERT/UPDATE/DELETE 事件，实时同步到数仓/搜索引擎/缓存。工具：Debezium（开源主流，支持 MySQL/PG/MongoDB）→ Kafka → Flink ETL → 目标端。优势：解耦上游业务写入，不侵入业务库；挑战：schema 变更兼容/大事务延迟/幂等写入

---

## 5. 数据治理

- **数据血缘自动获取** — 解析 SQL/ETL 任务中的 source table→target table 映射（Spark/Atlas Lineage Hook 监听 RDD/DataFrame 的血缘关系）。变更影响评估时沿血缘反向追踪所有被消费该表的下游应用/报表/API
- **元数据管理** — 技术元数据（表结构/分区信息/格式/统计信息/owner/创建者）+ 业务元数据（指标口径/维度和度量含义/业务域分类）。数据资产目录（Atlas/DataHub）→ 检索数据资产 → 查看数据质量/字段含义/使用热门度
- **数据质量监控** — 六维监控：完整性（非空比例）、唯一性（是否重复）、一致性（跨表数据是否一致引用）、及时性（数据产出是否按 SLA 时间）、准确性（与真实数据源比对）、有效性（格式/范围约束）。监控规则配置在数据任务产出后立即校验，失败则阻断下游并告警
- **数据平台可观测性** — 任务级：调度延迟/任务失败率/资源利用率；数据级：行数波动/数据新鲜度/SLA 达成率；成本级：计算资源成本/存储增长趋势/资源浪费率（空闲集群/过期未删数据）。工具：DataHub/Atlas 元数据 + Grafana/自定义 Dashboard 统一视图