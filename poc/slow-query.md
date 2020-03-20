---
description: 戚铮  2020 年 3 月 1 日
---

# 慢 SQL 的分析和优化

## **一、背景**

当线上运行的数据库性能变差或出现抖动时，我们的直观感受是 SQL 响应缓慢、监控 Duration 升高、业务出现超时等。SQL 响应慢除了 SQL 本身执行效率的原因，往往还会受到集群内外各个组件、环境、人为等因素的影响。如果只考虑集群内部因素，建议按照 [Performance Map](https://github.com/pingcap/tidb-map/blob/master/maps/performance-map.png) 的读写流程逐一进行排查，比如先确认慢在 TiDB 还是 TiKV，再看 TiKV 内部各组件的延迟情况，是否 Region 分布不均或有读写热点，某个组件或资源是否达到瓶颈，是否有堆积排队或报错重试，线程池内部是否负载均衡，关注 RocksDB 的读写延迟、缓存命中率变化等，以及系统资源消耗、 CPU 负载、IO 吞吐和耗时，网络带宽和延迟是否出现异常现象等。

如果日常业务运行比较稳定，QPS 和延迟维持在合理的基线上，并且没有外在变更和环境等因素的干扰，当数据库性能抖动，很可能是某些 SQL 的执行计划出现偏差，往往表现为选错了索引，或有索引但没用上，选错了 Join 方式比如该用 Index Join 却用了 Hash Join 或 Merge Join，以及 Join 的顺序不合理等。此时我们可以通过排查慢查询等方式定位 SQL，分析变慢的原因并避免问题复现。

另外新业务刚上线或已有业务从 MySQL 迁移到 TiDB，通常会先对数据库进行压测，包括性能测试，评估 SQL 延迟能否达到业务要求。当 SQL 执行时间不符合预期，或者与其他数据库差距较大时，我们需要了解慢 SQL 的常见原因、处理思路、新版本 TiDB 优化器相关的改进，尝试调整 SQL 执行计划，对慢 SQL 进行优化。 


## **二、预备知识**

在排查慢 SQL 问题前，需要对相关知识有一定的了解，建议先阅读官方文档，主要包括

* [SQL 优化流程](https://pingcap.com/docs-cn/stable/reference/performance/sql-optimizer-overview/)
* [执行计划](https://pingcap.com/docs-cn/stable/reference/performance/understanding-the-query-execution-plan/)
* [统计信息](https://pingcap.com/docs-cn/stable/reference/performance/statistics/)
* [慢查询日志](https://pingcap.com/docs-cn/stable/how-to/maintain/identify-abnormal-queries/identify-slow-queries/)
* [优化器 Hint](https://pingcap.com/docs-cn/stable/reference/performance/optimizer-hints/)
* [执行计划绑定](https://pingcap.com/docs-cn/stable/reference/performance/execution-plan-bind/)
* [expensive query](https://pingcap.com/docs-cn/stable/how-to/maintain/identify-abnormal-queries/identify-aborted-queries/)
* [Statement Summary](https://pingcap.com/docs-cn/stable/reference/performance/statement-summary/)


## **三、慢 SQL 的分析和优化**

### **3.1 索引选择**

优化器会为可用的每一个索引计算相应的 cost，并且选择 cost 最小的索引，使用不同的索引可能会有不同的 key range，读取不同的数据量，也就是 explain sql 结果中对应行的 count 值，目前可以认为这个估算的 row count 值就是索引的 cost。

如果执行计划选错了索引或没有选择索引，很大可能是 row count 估算不正确，常见原因有

* 统计信息过期不准确，需要改进收集统计信息的方式，维护统计信息的准确度，详见本文处理思路。
* 数据分布的差异性较大，统计信息的精确度不够，导致优化器做 row count 估算时存在独立性假设、均匀分布假设失效的情况，可以通过调整优化器参数如 
`tidb_opt_correlation_threshold`，`tidb_opt_correlation_exp_factor` 影响 row count 的估算结果。
* 统计信息 CMSketch hash 冲突导致，比如当某一行和另一个频次很高的行映射到了相同的 cell 里，可能导致索引 cost 估算过高进而无法选择索引，3.0.6 版本将 CMSketch 中出现次数最多的 N 个值抽取出来，提高了估算准确度。

当慢查询日志中没有 `index_ids` 或 `index_name` 字段，或者查询 `slow_query` 表的上述 index 字段为空，就表示查询没有用到任何索引，需要关注这类 SQL，特别是扫了大量 keys 的查询会占用更多的资源。如果没有使用索引并不代表执行计划不合理，比如小表扫描或先走索引再大量回表查询的 SQL，可能 TableScan 效率更高。

一般情况下，通过评估列的选择性，来判断列上是否适合建索引

* 计算列的非重复记录数与表的总记录数的比值，即 count(distinct column) / count(*) 结果来越接近 100 ，说明选择性越好，更适合建索引。
* 如果存在多个过滤字段、关联字段、排序字段等，尽量使用组合索引，避免单索引的回表操作，组合索引中将选择好的过滤字段放在前面。
* 按需创建索引，避免冗余和未使用的索引，索引过多对增删改有性能开销。
* 创建索引后，使用 explain 检查 SQL 是否合理使用索引。

### **3.2 Join 算子**

支持的 Join 算子

* Index Lookup Join：读取外表的数据，并对内表进行主键或索引键查询。
* Hash Join：将参与连接的小表预先装载到内存中，读取大表的所有数据进行连接。
* Sort Merge Join：利用输入数据的有序信息，同时读取两表的数据并依次进行比较。

Join 中的 Inner 表和 Outer 表

* Left [Outer] Join：左表是 Outer 表，右表是 Inner 表
* Right [Outer] Join：右表是 Outer 表，左表是 Inner 表
* Inner Join：Outer 和 Inner 表由优化器根据 cost 选择

Join 适用场景

* 如果 Join Key 比较少，而 Inner 表的数据量比较大，Outer 表的数据量比较小，那么大部分情况下使用 Index Join 会比其他 Join 算法更快，并且消耗更少的资源。Index Join 相关参数有 `tidb_index_join_batch_size` 和 `tidb_index_lookup_join_concurrency`。
* 如果 Join Key 比较多，大部分情况下直接使用 Hash Join 会比较快，它会把 Inner 表的数据全部拿上来构建 Hash 表，所以如果 Inner 表的数据量过大，可能会 OOM。Hash Join 相关参数有 `tidb_hash_join_concurrency`。
* 如果数据量比较大，而 Inner 表和 Outer 表各自的 Join Key 又能匹配到某个索引的前缀，那么这时候可以使用 Sort Merge Join 来避免 OOM。不过需要注意的是，Merge Join 会把同一个 Join Key 的数据 buffer 在内存中，如果某个 Join Key 的数据量特别大，也可能 OOM。

默认优化器会基于当前统计信息选择一个它认为最优的 Join 算法，此外也提供了相应的优化器 Hint 控制选择不同的 Join 方式。关于 Join 算法原理详见源码阅读文章 [Index Lookup Join](https://pingcap.com/blog-cn/tidb-source-code-reading-11/)，[Hash Join](https://pingcap.com/blog-cn/tidb-source-code-reading-9/)，[Sort Merge Join](https://pingcap.com/blog-cn/tidb-source-code-reading-15/)。

### **3.3 Join 顺序**

当执行多表 Join 时，优化器会选择一个它认为合适的 Join Order。一般的规则是，将小表或 Join 后较小的结果集放在前面做 Join 比较好，比如一种情况是当 Join Key 是等值条件且相关列有 Unique 属性，那么 Join 后只返回一行记录，将这类 Join 放在前面优先处理，通常不会造成结果集的膨胀。

如果 Join Order 没有选好，可能出现中间某些 Join 结果集非常大，后面的 Join 需要处理大量数据，导致整个 SQL 执行慢；这时可以考虑手动选择 Join Order，将产生一个更小的结果集的 Join Order 提前；当确定好顺序后，可以用 `straight join` 对 Join Order 进行固定。

### **3.4 聚合算子**

支持的聚合算子

* Stream Aggregate：要求对输入数据按照 group key 有序。
* Hash Aggregate：对输入数据无顺序要求，根据输入数据的 group key 构造 hash 表，* hash aggregation 算法 partial 阶段和 final 阶段可以调整并发参数 `tidb_hashagg_partial_concurrency` 和 `tidb_hashagg_final_concurrency`。

如果数据按照 group key 有序，Hash Aggregate 在不使用并发的情况下，一定比 Stream Aggregate 要慢。对于 Hash Aggregate 来说，如果 group key 特别多，可能会导致 OOM。Stream Aggregate 通常占用更少的内存，但执行时间更长。在数据量太大或系统内存不足时，建议尝试使用。4.0 版本支持使用 `HASH_AGG` 和 `STREAM_AGG` Hint 选择不同的聚合算子。关于 Hash Aggregate 算法原理详见 [Hash Aggregation](https://pingcap.com/blog-cn/tidb-source-code-reading-22/)。

### **3.5 处理思路**

如果不确定是否执行计划偏差导致 SQL 执行慢，可以关注慢查询中记录的相关耗时，其中 Query_time 字段表示 SQL 执行消耗的总时间，包括 `Parse_time`/`Compile_time`/`Backoff_time`/`Process_time`/`Wait_time`，其中 `Process_time` 是真正用于处理计算和查询的时间，通常和 Coprocessor 扫描的 `Total_keys` 和 `Process_keys` 数量相关；其他时间如 parse/compile/backoff/wait 正常时应接近 0，集群负载过高可能会导致这些时间上涨，但是与 `Query_time` 占比不应过大。

如果慢查询中的耗时看上去都比较正常，还可以通过 explain analyze 和 [trace](https://pingcap.com/docs-cn/stable/reference/sql/statements/trace/) 真实执行查询 SQL，确认各阶段处理的耗时情况。当确认是执行计划不合理导致的 SQL 慢，需要继续进行排查，如统计信息是不是准确，更新统计信息后有没有改善，优化器 cost 估算是否有偏差，尝试通过加 Hint，调整参数等方式优化。

**检查统计信息是否准确** 

* 执行 `show stats_meta`，查看表的总行数 count 以及修改的行数 `modify_count`，`modify_count` 过大时表示数据更新频繁，说明统计信息可能不准确了。
* 执行 `show stats_healthy` ，当 healthy 小于 50 会触发统计信息自动收集（`run-auto-analyze` 默认 true，`tidb_auto_analyze_ratio` 阈值默认 0.5）；当 healthy 大于 50 也不意味着统计信息没问题，可能存在表数据分布严重不均或某一段数据频繁更新的情况，在不影响业务的前提下，建议设置收集的时间窗口，调大 `tidb_auto_analyze_ratio` 参数，增加收集频次。
* 当 `modify_count/count` 超过一定阈值（目前是 0.7）优化器会使用 pseudo 的统计信息，pseudo 的统计信息无法反映真实的数据分布，大概率会让优化器选择错误的执行计划；如果执行计划中 TableScan/IndexScan 这一行显示 stats:pseudo，说明 table 的统计信息过期，应及时收集统计信息。
**注意：**在 Index Lookup Join 场景下，如果 TableScan/IndexScan 上面有 Selection，那么 TableScan/IndexScan 这一行出现 stats:pseudo 显示是正常的，实际使用了统计信息，后面版本会 fix 这个问题。
* 通常情况下，执行计划不会出现 pseudo ，统计信息也不应该有 healthy 远低于 `(1 - tidb_auto_analyze_ratio) * 100` 的情况；如果出现，查看日志中的 auto analyze triggered 关键字，是否存在异常情况导致自动收集没有运行。
**注意：**3.0.8 版本之前，`tidb_auto_analyze_ratio` 的最小值代码中限制为 0.3，后面的版本移除了这个限制。
* 执行 explain analyze 耗时可以接受的情况下，对比结果中 count 和 execution info 中 rows 的差距，当 TableScan/IndexScan 这一行差距较大时，很可能是统计信息不准了；如果 explain analyze 耗时过长，可以通过对 TableScan/IndexScan 上的条件执行 select count(*)，与 explain analyze 中的 row count 对比，确定是否统计信息问题。


**手动收集统计信息**

* 收集前建议先导出统计信息保留现场，便于后续问题排查，导出命令 `curl -G "http://${tidb-server-ip}:${tidb-server-status-port}/stats/dump/${db_name}/${table_name}" > table_name_stats.json`
* 执行 analyze table，analyze 会切分为多个子任务，每个任务只负责某一个列或者索引，`tidb_build_stats_concurrency` 控制同时执行的任务的数量，默认为 4。
* 对几十亿数据的大表收集执行时间较长，可能会对其他 SQL 造成一定影响，需要评估是立即收集还是等业务低峰期，可以考虑
	* 调整 `tidb_build_stats_concurrency` 加快收集速度或减小线上影响
	* 启用 fast analyze 快速收集
	* 只对相关索引进行收集或对索引执行增量收集
* `show analyze status` 检查 analyze 进度，完成后 explan sql 确认，如果优化有效果，通过 crontab 定期 analyze table 或者索引；如果效率没有改善，尝试加 Hint 调整执行计划。


**添加优化器 Hint**

* 使用 `use index(index_name/primary)` 强制优化器使用正确的索引，也支持 force index，目前这两者语义上没有做区分。
* 使用 `ignore index(index_name)` 强制优化器不使用指定的索引。
* 使用 `TIDB_INLJ`、`TIDB_HJ`、`TIDB_SMJ` 指定 Join 方式。
* 如果不希望修改 SQL，可以使用执行计划绑定的方式，使用 `create binding` 创建绑定，将原始 SQL 绑定为加 Hint 后的 SQL。
* 对于某些执行计划异常变动性能变慢的查询或 expensive query，有可能影响整个集群性能，通过加 Hint `max_execution_time(ms)` 或设置变量方式指定执行预期时间，超时中止。


**调整性能相关参数**

* Table/Index Scan 相关参数
	* `tidb_index_lookup_size`
	* `tidb_index_lookup_concurrency`
	* `tidb_index_serial_scan_concurrency`
	* `tidb_distsql_scan_concurrency`
* 算子相关参数
	* `tidb_index_join_batch_size`
	* `tidb_index_lookup_join_concurrency`
	* `tidb_hash_join_concurrency`
	* `tidb_projection_concurrency`
	* `tidb_hashagg_partial_concurrency`
	* `tidb_hashagg_final_concurrency`
* 优化器相关参数
	* `tidb_opt_insubq_to_join_and_agg`
	* `tidb_opt_agg_push_down`
	* `tidb_opt_correlation_threshold`
	* `tidb_opt_correlation_exp_factor`

关于参数说明详见 [TiDB 专用系统变量和语法](https://pingcap.com/docs-cn/stable/reference/configuration/tidb-server/tidb-specific-variables/)


**使用 TiFlash 引擎**

在很多分析场景下，即使 TiDB 优化器已经能够选择合适的执行计划，且扫描任务也都下推到各个 TiKV 节点，但是 Coprocessor 处理大量数据仍然要花费较长时间，通过调整参数等优化手段后的改善可能并不明显。建议测试使用 TiFlash 列存处理这些查询，通常能够让执行效率获得成倍的提升。TiFlash 测试结果详见 [如何轻松让 TiDB 分析场景无痛提速十倍](https://asktug.com/t/topic/2626)。


**反馈到 TiDB 官方**

在收集统计信息后，如果仍无法选择较优的执行计划，建议关注各版本 Release Notes 中 SQL 优化器相关改进，并考虑升级到新版本。如果仍有问题，欢迎反馈到 AskTUG 或技术支持人员，并提前收集相关信息，包括

* TiDB 版本信息
* CREATE TABLE 建表语句和慢查询 SQL
* 慢查询日志和 `tidb_decode_plan` 输出的执行计划，3.0.5 版本慢查询默认记录 Plan 信息，通过 `select tidb_decode_plan('xxx')` 查询
* 导出问题出现时相关表的统计信息


## **四、相关案例**

**Asktug 问题**

* [https://asktug.com/t/tidb/1433](https://asktug.com/t/tidb/1433)
* [https://asktug.com/t/count/1516](https://asktug.com/t/count/1516)
* [https://asktug.com/t/topic/2577](https://asktug.com/t/topic/2577)
* [https://asktug.com/t/topic/2652](https://asktug.com/t/topic/2652)
* [https://asktug.com/t/topic/2705](https://asktug.com/t/topic/2705)
* [https://asktug.com/t/topic/2714](https://asktug.com/t/topic/2714)
* [https://asktug.com/t/topic/2748](https://asktug.com/t/topic/2748)


