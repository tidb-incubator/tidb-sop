---
description: 戚铮  2020 年 3 月 10 日
---

# 上线前注意点

## **一、背景**

在分布式数据库产品选型调研期间，用户通常会对 TiDB 进行各种测试，如功能测试、性能测试、可用性测试、稳定性测试等，评估集群是否满足业务需求，承载峰值流量，并具备高可用和稳定的性能表现。

评估测试通过后，在系统正式上线或割接前，DBA 或运维人员可以对集群做一次健康检查，确认软硬件环境、配置参数满足运行要求，常见监控项没有异常，相关日志没有报错，上线后应关注数据库延迟、QPS 是否符合预期，重要监控指标有没有告警等，确保集群在业务周期内正常运行。


## **二、上线前 Check 项**

### **2.1 系统环境**

建议按照推荐的 Ansible 方式部署，执行 bootstrap 和 deploy 步骤时会修改相关参数，检查环境是否满足要求；如果跳过相关步骤，或未使用 ansible 方式部署，则需要单独进行额外检查，避免系统环境因素对集群上线后的运行造成影响。

**内核版本**

目前版本的 gRPC 默认使用 epollex engine，它使用了一个较新的内核特性 EPOLLEXCLUSIVE ，对于 centos 来说需要 3.10.0-862.14.1.el7 以上的内核支持。

如果内核不支持，gRPC 会自动使用性能较差的 epoll1 engine，内部测试结果显示性能下降 20% 左右。

**NUMA**

上线时使用的 x86 物理服务器通常都是多路架构，2U 服务器打开超线程后 CPU vcore 数量通常在 40 以上，少量 4U 服务器超线程可能超过 64 vcore。TiDB Server 使用 Go 语言开发，golang 本身存在 gc 开销，单个 tidb-server 进程在 CPU 核数较多的情况下 performance 表现可能不高，也会引起频繁的上下文切换导致 system cpu 消耗较高。

如果是典型的 TP 业务，tidb-server 对于内存的消耗不高，出现 OOM 的风险较低；在 vcore 较多的机器上建议单机部署多个 TiDB 实例，并结合手工 numa 绑核来提升 user cpu 的使用率；通常分配给单个 tidb-server 的 CPU core 建议在 16~24，并按照这个比例的 80% 设定 tidb-server 的 `max_proc` 配置项。

**THP**

THP 是 Transparent Hugepage 的简写，作为 Linux 内存管理系统中的一种优化特性，通过使用较大的内存页面来降低大内存服务器环境下的 TLB 查询开销。当前 THP 只支持匿名页面，在某些场景下  Linux 内核会将大页面分裂成小页面来满足内存分配需求，而这个操作会降低系统性能，并增加内存碎片。

系统运行期间，在持续的内存压力下，动态打开或关闭 THP 会触发大页小页之间的转换等操作，也会导致内存碎片过多，影响系统的性能和稳定性。
建议在系统启动时就关闭 THP，或在线关闭 THP 后重启系统。

**时间同步**

机器所在的内网环境通常提供了时间同步服务如 NTP、Chrony，应检查相关服务运行正常且配置正确。如果网络中没有 NTP Server 等，需要选择一个节点作为 Server 单独部署。

**防火墙**

建议默认关闭系统防火墙。如果内网环境限制了网络访问策略，则需要申请为相关[服务端口](https://pingcap.com/docs-cn/stable/how-to/deploy/hardware-recommendations/#%E7%BD%91%E7%BB%9C%E8%A6%81%E6%B1%82)添加白名单，开放端口访问权限。

**fio 测试**

执行 bootstrap 时 Ansible 会用 fio 命令对磁盘性能进行检测，具体测试命令位于 tidb-ansible/roles/machine_benchmark/tasks 目录下的 yml 文件中，建议使用本地 SSD 盘，相关指标的标准值参照下表

指标 | 标准值
-|-
随机读与顺序写混合测试 / read iops | 不低于 10000 |
随机读与顺序写混合测试 / read latency (ns) | 不高于 250000 |
随机读与顺序写混合测试 / write iops | 不低于 10000 |
随机读与顺序写混合测试 /write latency (ns) | 不高于 30000
随机读测试 / read iops | 不低于 40000

使用云主机部署时，由于机型差异，测试结果可能达不到标准值要求，但不应该差距过大，特别是高写入场景，尽量避免上线后磁盘成为性能瓶颈。

**网络测试**

ping 延迟测试，通常同机房机器之间网络延迟应小于 1ms，跨机房之间的延迟应小于 5ms，也可以在部署集群后通过 blackbox exporter 监控看到实际的延迟情况。

内网带宽测试，部署建议使用万兆网卡，但带宽要达到 10Gbps 还需要万兆交换机支持，可以使用网络性能测试工具如 qperf、iperf 测试内网实际带宽。

### **2.2 集群信息**

**版本信息**

目前线上集群推荐使用 3.0.X 的稳定版本，未来 3.1 或 4.0 版本 GA 后也可部署使用；避免使用 lastest 或 master 版本，这些版本通常在持续开发中，没有跑过正式的发版测试流程，存在未知风险。检查各个组件的版本是否一致，避免版本不一致可能导致的兼容性问题。

**inventory 配置**

TiKV 单机多实例部署时，需要检查 labels 设置是否正确，相同机器的端口如 tikv_status_port 是否冲突等；必须同时配置 PD 的 location-labels 和 TiKV 的 labels 参数，否则 labels 不会生效。**注意：各个版本的 inventory.ini 文件，以及 conf 目录下的配置文件模板不保证一致，应该在当前 ansible 目录下的模板文件基础上进行修改，避免从其他版本的目录下直接拷贝。**

**监控状态**

* Overview Dashboard

	* Services Port Status：各服务在线节点数量，确认没有 down 掉的节点。
	* System Info：没有业务压力时，各节点的 CPU、Memory、IO 负载和网卡流量应维持在较低水位线且没有明显波动。

* PD Dashboard
	* Store Status：集群各 TiKV 节点的状态，正常情况都是 Up Stores 的节点。 
	* Region Health：每个 Region 的状态；通常 pending 的 peer 应该少于 100，miss 的 peer 不能一直大于 0。

* TiDB Dashboard
	* Connection Count：连接数和业务相关，正常情况单个 tidb server 的连接数在1k 以下比较合适，且没有明显跳变。
	* Goroutine Count：业务连接数小于 1k 时，tidb 的 goroutine 数量通常在 1k~10k 之间，如果数量超过 10k 或持续增加，可能存在异常。
	* Failed Query OPM：执行 SQL 语句发生错误的统计，正常情况不应该出现。
	* Internal SQL OPS：内部 SQL 语句执行数量统计，如收集统计信息的 SQL 等，正常情况不应该过高。
	* Uncommon Error OPM：TiDB 非正常错误的统计，包括 panic，binlog 写失败，正常情况不应该出现。
	* TiClient Region Error：TiKV 相关的错误信息，如 not leader 等，数量不多属于正常现象。
	* KV Backoff OPS：TiKV 返回错误信息的数量，如事务冲突等，如果偶尔出现且数量不多可以忽略。

* TiKV Dashboard
	* Server is busy：各种会导致 server 繁忙的事件个数，如 write stall，channel full 等，正常情况下应该为 0。
	* Server report failures：server 报错的消息个数，正常情况下应该为 0。
	* Leader drop：TiKV drop leader 的个数，偶尔出现属于正常的调度行为。
	* Leader missing：TiKV  missing leader 的个数，正常情况下应该为 0。
	* Thread CPU：空负载下的各 Thread CPU 使用率，如 raft store、async apply、scheduler、coprocessor 等应该都比较低。

**日志信息**

* 检查系统日志，如 /var/log/message & kern、dmesg 是否存在系统内核等异常报错。
TiDB 各组件日志默认位于 {deploy_dir}/log 目录下，如 tidb.log，除了连接中断和偶尔的 SQL 执行报错外，日志中不应该出现大量 Error 关键字的报错信息。另外如果出现进程 panic 等异常，会打印到 stderr.log 结尾的错误日志中。

**统计信息**

* 检查统计信息是否过期，执行 show stats_meta、show stats_healthy，如果 modify_count 比较大或 healthy 不高，执行 analyze 收集统计信息。

* 默认自动 analyze 已打开， analyze ratio 为 0.5；对于更新比较频繁的表，建议提高收集频次，调小 analyze ratio 并设置时间窗口，例如
	* set global tidb_auto_analyze_ratio=0.2;
	* set global tidb_auto_analyze_start_time='00:00 +0800';
	* set global tidb_auto_analyze_end_time='06:00 +0800';

**热点表**

检查所有的业务表，是否存在 AUTO_INCREMENT 自增主键，或单调递增的索引如时间类型字段的索引。

* 对于主键非整型或没有主键的表，建议设置 SHARD_ROW_ID_BITS，将 rowid 打散到不同的 region，缓解写热点。
* 对于 AUTO_INCREMENT 自增主键的表，在设置 SHARD_ROW_ID_BITS 之前，需要先去掉自增主键，如果业务依赖自增主键，考虑将主键改为自增唯一键（在TiDB 中，自增列只保证自增且唯一，并不保证连续分配），但仍无法避免索引写热点。
* 对于业务主键为整型的表，如果写入时主键非单调递增，不会导致热点问题；如果是顺序递增写入，可以用 hash partition 打散热点，需要调整业务代码；或者将整型主键与其他列改造为联合主键，应用端选出合适的多列（不重复、写入时数据的分布有较好的随机性）作为主键。
* 对于需要开启 tidb-binlog 的集群，为保证同步效率和数据一致性，会强制要求表上有主键或唯一键，如果业务没办法选出主键或唯一键，最终还是会用自增列做主键/唯一键；对于这个需求，4.0 版本支持用 auto_random 自动生成打散的整型主键，避免了唯一键的索引热点。
* 对于小表或固定范围内的热点表，可以用 split region 将热点打散到多个 region 上；对于读热点，4.0 的 follower read 也能够起到分散热点的作用。


## **三、上线后关注点**

### **3.1 监控告警**

重要业务对集群的可用性和稳定性要求较高，建议上线前部署 alert manager 告警组件，并配置好相应的电话、短信等通知接口，可以根据 prometheus 的重要监控 metric 自定义告警规则和阈值，上线后及时关注重要告警信息。

**TiDB 告警**

* TiDB_schema_error
	* 含义：TiDB 在一个 Lease 时间内没有重载到最新的 Schema 信息, TiDB 无法继续对外提供服务。
	* 处理办法：通常因为 TiKV region unavailable 或超时，需要看 TiKV 的监控项定位问题。

* TiDB_tikvclient_region_err_total
	* 含义：TiDB 访问 TiKV 发生 Region 错误, 在 10 分钟内多于 6000 次则告警。
	* 处理办法：检查 TiKV 的状态是否正常。

* TiDB_domain_load_schema_total
	* 含义：TiDB 重载最新的 Schema 信息失败的总次数，如果在十分钟之内重载失败次数超过了 10 次则报警。
	* 处理办法：通常因为 TiKV region unavailable 或超时，需要看 TiKV 的监控项定位问题。

* TiDB_monitor_keep_alive
	* 含义：表示 TiDB 的进程是否存在，如果在十分钟内 metric 增加次数少于100, 表示 TiDB 的进程可能退出了。
	* 处理办法：检查 TiDB 进程是否 OOM 或机器是否发生了重启。

* TiDB_server_panic_total
	* 含义：表示发生 panic 的总数。
	* 处理办法：检查 log 中 panic 相关信息。

**PD 告警**

* PD_cluster_offline_tikv_nums
	* 含义：PD 长时间（默认配置是 30 分钟）没有收到 TiKV 心跳。
	* 处理办法：检查 TiKV 进程是否正常，网络是否隔离，是否负载过高，尽可能恢复服务；如果确定 TiKV 无法恢复，可做下线处理。

* PD_etcd_write_disk_latency
	* 含义：etcd 写盘耗时过长，可能是 IO 延迟较高。
	* 处理办法：检查是否混合部署导致 IO 性能不足。

* PD_miss_peer_region_count
	* 含义：缺失副本的 region 数量超过 100。
	* 处理办法：检查是否有 tikv 宕机或是否有相关 bug。

**TiKV 告警**

* TiKV_memory_used_too_fast
	* 含义：TiKV 内存占用增长过快，默认规则 5 分钟内存使用超过 5GB 会告警。
	* 处理办法：通过 node_exporter 监控集群内机器的内存使用情况，2.1 版本调整 rockdb.defaultcf 和 rocksdb.writecf 的 block-cache-size 的大小；3.0 版本调整 block-cache 的 capacity 大小。

* TiKV_GC_can_not_work
	* 含义：6 小时内没有成功执行 GC 的 region，说明 GC 不能正常工作了。短期内 GC 不运行不会造成太大的影响，但是如果 GC 一直不运行，会导致版本越来越多，从而导致查询变慢。
	* 处理办法：检查 gc leader 所在的 tidb server 日志，如果一直在 resolve locks 或者 delete ranges 说明是正常现象，否则应联系技术支持人员处理。

* TiKV_server_report_failure_msg_total
	* 含义：给特定 store 发消息失败10分钟内超过10次。
	* 处理办法：查看网络是否异常。

* TiKV_channel_full_total
	* 含义：channel full 个数10分钟内超过10次，持续的 channel full 会影响业务。
	* 处理办法：检查集群负载和相关组件日志。

* TiKV_write_stall
	* 含义：RocksDB 写入出现阻塞。
	* 处理办法：查看 tikv 监控面板 scheduler pending commands 是否持续超过阈值，是的话是有写堆积，需要排查是否写入压力过大。

* TiKV_raft_log_lag
	* 含义：副本 raft log 落后情况，如果在 1% log lag bucket 在 1min 内超过了 5000 会报警。
	* 处理办法：检查磁盘和网络使用情况。

* TiKV_async_request_write_duration_seconds
	* 含义：异步 write 时间，1分钟内有 1% 的请求大于 1s 报警。
	* 处理办法：检查告警节点的 IO 资源是否成为瓶颈。

* TiKV_coprocessor_request_wait_seconds
	* 含义：读请求等待时长 1 分钟内有 1% 的等待时间超过了10s。
	* 处理办法：查看慢 SQL 日志进行问题定位。

* TiKV_raftstore_thread_cpu_seconds_total
	* 含义：1分钟内 raftstore 线程 CPU 使用率超过了 80%，集群整体表现卡顿。
	* 处理办法：2.1 版本扩容 tikv 实例，3.0 版本调大 raftstore 线程数。

* TiKV_raft_append_log_duration_secs
	* 含义：1分钟内 1% 的日志写 raft log 所需时间超过了 1s。
	* 处理办法：告警服务器的 IO 或者 CPU 成为瓶颈。

* TiKV_raft_apply_log_duration_secs
	* 含义：1 分钟内 1% 的日志 apply log 所需时间超过了 1s。
	* 处理办法：告警服务器的 IO 或者 CPU 成为瓶颈。

* TiKV_scheduler_latch_wait_duration_seconds
	* 含义：KV command 等待内存锁的时间。
	* 处理办法：查看是否存在写入堆积。

* TiKV_thread_apply_worker_cpu_seconds
	* 含义：一分钟内 apply worker 线程 CPU 使用率高于 90%。
	* 处理办法：2.1 版本扩容 tikv 实例，3.0 版本调大 raftstore 线程数。

* TiDB_tikvclient_gc_action_fail
	* 含义：tikv 客户端 gc 失败。
	* 处理办法：修改 tikv_gc_auto_concurrency 并行度，提升 gc 处理效率。


### **3.2 日志报错**

**TiKV server is busy**

* raftstore is busy: 可能由于写入热点或 RocksDB 写入太慢，结合监控和日志判断。
* coprocessor full: coprocessor 任务堆积，可能是由于读热点或 RocksDB 读取太慢，如果 Coprocessor CPU 负载较高，可以调大 参数readpool.coprocessor.max-tasks-per-worker-normal、max-tasks-per-worker-high、max-tasks-per-worker-low 缓解。
* scheduler is busy: scheduler 中由于等待 latch 而阻塞的请求大小超出上限，参考 [diagnose-map](https://github.com/pingcap/tidb-map/blob/master/maps/diagnose-map.md) 处理。

**Write conflict**

事务冲突报错，与业务逻辑有关，结合 TiDB 监控项 KV Retry Duration、Lock Resolve OPS、KV Backoff OPS 定位。

**locked primary_lock**

读写冲突，读数据时发现 key 有锁阻碍读，少量报错不需要处理。

**request outdated**

请求执行时间过久被取消，说明有超大请求，或有大量请求积压；若业务侧的确需要执行超大请求，可调大 TiKV 请求执行时间限制（[end-point-request-max-handle-duration](https://github.com/tikv/tikv/blob/master/etc/config-template.toml#L152)）。

**TxnLockNotFound**

事务提交慢超过了 TTL 被其他事务给 Rollback 掉了，事务 ttl 默认是 3s，需要排查提交慢的原因，参考 [diagnose-map](https://github.com/pingcap/tidb-map/blob/master/maps/diagnose-map.md) TiKV 写入慢处理

**Information schema is changed**

详见[触发 Information schema is changed 错误的原因](https://pingcap.com/docs-cn/stable/faq/tidb/#3313-%E8%A7%A6%E5%8F%91-information-schema-is-changed-%E9%94%99%E8%AF%AF%E7%9A%84%E5%8E%9F%E5%9B%A0)

**Information schema is out of date**

详见[触发 Information schema is out of date 错误的原因](https://pingcap.com/docs-cn/stable/faq/tidb/#3313-%E8%A7%A6%E5%8F%91-information-schema-is-changed-%E9%94%99%E8%AF%AF%E7%9A%84%E5%8E%9F%E5%9B%A0)

**GC life time is shorter than transaction duration**

SQL 语句执行时间过长，超过了 MVCC 的版本 GC 时间，调大 gc_life_time。

**txn takes too much time**

调大 gc_life_time 以及 [max-txn-time-use](https://github.com/pingcap/tidb/blob/release-3.0/config/config.toml.example#L291)

**Region is unavailable**

参考 [diagnose-map](https://github.com/pingcap/tidb-map/blob/master/maps/diagnose-map.md) 处理

### **3.3 慢查询**

**常见原因**

* 执行计划偏差
* 大查询，AP 场景
* 读热点，热点 region 或热点 tikv
* 资源受限，如 CPU Coprocessor 线程不足、内存 Block Cache 较小、跨机房网络延迟较高、普通 SSD 延迟较高等

**排查途径**

* 分析监控 metrics 如 TiKV Coprocessor CPU 是否过高，详见 [Coprocessor 监控指标](https://pingcap.com/docs-cn/stable/reference/key-monitoring-metrics/tikv-dashboard/#coprocessor)
* 分析 slow query log，支持 pt-query-digest 工具，或者根据 slow_query 内存表，自定义查询需求，详见[定位慢查询](https://pingcap.com/docs-cn/stable/how-to/maintain/identify-abnormal-queries/identify-slow-queries/)
* 对于分析场景的大查询，消耗资源较多，详见 [expensive query 日志](https://pingcap.com/docs-cn/stable/how-to/maintain/identify-abnormal-queries/identify-aborted-queries/#%E5%AE%9A%E4%BD%8D%E6%B6%88%E8%80%97%E7%B3%BB%E7%BB%9F%E8%B5%84%E6%BA%90%E5%A4%9A%E7%9A%84%E6%9F%A5%E8%AF%A2)
* 根据 tikv 日志关键字 slow-query 过滤出的 table_id 或 start_ts 排查 tidb slow query 日志
* 3.0 版本提供 [statement summary table](https://pingcap.com/docs-cn/stable/reference/performance/statement-summary/#statement-summary-tables)，对同类 SQL 分组并按照延迟、执行次数等多维度统计，帮助定位历史 TopSQL


### **3.4 常见问题及处理**

* [TiDB 诊断地图](https://github.com/pingcap/tidb-map/blob/master/maps/diagnose-map.md)
* [TiDB FAQ](https://pingcap.com/docs-cn/stable/faq/tidb/)
* [升级 FAQ](https://pingcap.com/docs-cn/stable/faq/upgrade/)
* [DM FAQ](https://pingcap.com/docs-cn/stable/reference/tools/data-migration/faq/)
* [TiDB Binlog FAQ](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/faq/)
* [TiDB Binlog 故障诊断](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/troubleshoot/binlog/)
* [DM 故障诊断](https://pingcap.com/docs-cn/stable/reference/tools/data-migration/troubleshoot/dm/)
* [TiDB Lightning FAQ](https://pingcap.com/docs-cn/stable/faq/tidb-lightning/)


## **四、相关案例**

**Asktug 问题**

* [https://asktug.com/t/topic/1412](https://asktug.com/t/topic/1412)
* [https://asktug.com/t/topic/1747](https://asktug.com/t/topic/1747)
* [https://asktug.com/t/topic/2213](https://asktug.com/t/topic/2213)
* [https://asktug.com/t/topic/2244](https://asktug.com/t/topic/2244)
* [https://asktug.com/t/topic/2261](https://asktug.com/t/topic/2261)
* [https://asktug.com/t/topic/2445](https://asktug.com/t/topic/2445)
* [https://asktug.com/t/topic/2504](https://asktug.com/t/topic/2504)
* [https://asktug.com/t/topic/32850](https://asktug.com/t/topic/32850)
