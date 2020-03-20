# 07 如何在线开启/关闭 TiDB Binlog
> 梁启斌  2020 年 2 月 10 日


## 一、背景 / 目的
TiDB-Binlog 是一个用于收集 TiDB 的 增量数据，并提供数据同步、异步备份功能的工具。
TiDB-Binlog 支持以下场景：

- 数据同步：同步 TiDB 集群数据到其他数据库、消息中间件以及文件
- 数据备份和恢复：异步备份 TiDB 集群数据，同时可以用于 TiDB 集群故障时恢复在日常使用当中，会遇到之前未开启 binlog 的集群需要开启，以及开启 binlog 的集群需要关闭，并且希望对业务的影响降到最低。本文主要描述使用 ansible 的方式在线开关 TiDB-Binlog 的操作步骤以及注意点。

## 二、相关概念及原理解释

关于 TiDB binlog 的组件 Pump 、 Drainer 、binlogctl 工具 的介绍可以参考官方文档的介绍([TiDB Binlog 整体架构介绍](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/overview/))

## 三、操作前 Check 项

### 3.1 开启 TiDB Binlog 需要 check 的事项:

1. Pump 以及 Drainer 的配置是否符合官方的推荐配置，特别需要注意如果部署的服务器原来已经有其他服务在运行，则需要评估该服务器的负载是否能够满足增加 Pump 或者 Drainer 。

2. 检查磁盘空间，确认是否有足够空间来给 Pump 以及 Drainer 对日志数据进行落盘。（ Pump 主要是保存上游 TiDB 发送过来的 binlog ，Drainer 如果下游对应的是 file 的话会在本地保存为文件。）

   ```bash
   shell> df -h 
   ```

3. 端口预留，检查端口是否被占用，当前是否可用等。（ 默认端口号 Pump ：8250 、Drainer : 8249 ）
   
   ```bash
   shell> netstat -anp |grep {port_no}
   ```

### 3.2 关闭 TiDB Binlog 需要 check 的事项:

1. 下线主要针对永久（或长时间）不再使用该服务的场景，如果只是需要临时维护请使用 pause-pump 或 pause-drainer 命令来暂停 Pump 或者 Drainer 。

## 四、注意事项

1. 在开启 enable-binlog 运行 TiDB 时，需要保证至少一个 Pump 正常运行。
2. 需要保证同一集群的所有 TiDB 都开启了 binlog 服务，否则在同步数据时可能会导致上下游数据不一致。尤其注意 Binary 方式部署 TiDB，要注意统一设置 enable-binlog 参数。
3. 由于开关 TiDB Binlog 需要重启 TIDB server ，因此会导致当前正在执行的 SQL 发生异常。建议在业务低峰期或者把需要重启的 TiDB 从 LB 排除，把影响降到最低。

## 五、操作步骤

### 5.1 开启 TiDB binlog

#### 当前集群拓扑信息

组件 | 机器 (IP端口) | 数量
---- | ---- | ----
TiDB | 172.16.4.243:4807 、 172.16.4.242:4807 | 2
PD | 172.16.4.243:12779 、172.16.4.242:12779 、172.16.4.240:12779 | 3
TiKV | 172.16.4.239:27173 、172.16.4.240:27173 、172.16.4.238:27173 | 2

确认当前的集群里面的有没有开启 binlog 以及 Pump 和 Drainer 的情况

  ```sql
  
  mysql> show global variables like 'log_bin';
  +---------------+-------+
  | Variable_name | Value |
  +---------------+-------+
  | log_bin       | 0     |
  +---------------+-------+
  1 row in set (0.01 sec)
  
  mysql> show pump status;
  Empty set (0.02 sec)
  
  mysql> show drainer status;
  Empty set (0.03 sec)
  
  ```

调整后集群目标拓扑信息

组件 | 机器 (IP端口) | 数量
---- | ---- | ----
TiDB | 172.16.4.243:4807 、 172.16.4.242:4807 | 2
PD | 172.16.4.243:12779 、172.16.4.242:12779 、172.16.4.240:12779 | 3
TiKV | 172.16.4.239:27173 、172.16.4.240:27173 、172.16.4.238:27173 | 2
**Pump** |**172.16.4.242:8250 、172.16.4.241:8250** | **2**
**Drainer** |**172.16.4.239:8249** | **1**


#### 操作中的主要步骤

1. 修改 tidb-ansible/inventory.ini 文件 , 添加 Pump 的配置，开启 enable_binlog 开关。（注：下列只是 inventory.ini 部分内容，在于突出需要修改的部分，需要正常运行 tidb-ansible 需要完整的 inventory.ini 文件。）

   ```toml
   ## Binlog Part
   [pump_servers]
   172.16.4.241
   172.16.4.242
   
   [drainer_servers]
   
   ## Group variables
   [pd_servers:vars]
   # location_labels = ["zone","rack","host"]
   
   ## Global variables
   [all:vars]
   deploy_dir = /home/tidb/deploy_3.0.6
   
   ## Connection
   # ssh via normal user
   ansible_user = tidb
   
   cluster_name = test-cluster_3.0.6
   
   tidb_version = v3.0.6
   
   # process supervision, [systemd, supervise]
   process_supervision = systemd
   
   timezone = Asia/Shanghai
   
   enable_firewalld = False
   # check NTP service
   enable_ntpd = True
   set_hostname = False
   
   ## binlog trigger
   enable_binlog = True
   
   ```

2. 使用 ansible 部署 Pump 。(注： 如果之前已经部署过相同版本的 Pump 且 deploy_path 一致的话可以跳过此步骤 。)

   ```bash
   shell> ansible-playbook deploy.yml --tags=pump
   ```

3. 使用 ansible 启动 Pump 。

   ```bash
   shell> ansible-playbook start.yml --tags=pump
   ```

4. 确认 Pump 已经正常启动。(**必须确保至少有一台 Pump 状态已经正常才能进行下一步操作**)
   
   ```bash
    shell> bin/binlogctl -pd-urls=http://172.16.4.243:12779 -cmd pumps
    
    [2019/04/28 09:29:59.016 +00:00] [INFO] [nodes.go:48]   ["query node"] [type=pump] [node="{NodeID: 1.1.1.1:8250,   Addr: 172.16.4.241:8250, State: online, MaxCommitTS:   408012403141509121, UpdateTime: 2019-04-28 09:29:57 +0000   UTC}"]
   ```

5. 更新并重启 TiDB servers  
   
   ```bash
   shell> ansible-playbook rolling_update.yml --tags=tidb
   ```

6. 使用 binlogctl 获取 initial_commit_ts 。(注意：本文仅列出获取 initial_commit_ts 的操作，详细请参考 [initial_commit_ts 获取。](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/deploy/#第-3-步部署-drainer))

   ```bash
   shell> {tidb_ansible_path}/resources/bin/binlogctl -pd-urls=http://172.16.4.243:12779 -cmd generate_meta 
   
   INFO[0000] [pd] init cluster id 6569368151110378289 2018/06/21 11:24:47 meta.go:117: [info] meta: &{CommitTS:400962745252184065}
   ```

7. 修改 tidb-ansible/inventory.ini 文件 , 添加 Drainer 的配置。（注意：本文以下游为 file 为例，如果下游为 MySQL 或者 Kafka 参考 [Drainer 部署](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/deploy/#第-3-步部署-drainer) ）
   
   ```toml
   [drainer_servers]
   drainer_file ansible_host=172.16.4.239 initial_commit_ts="400962745252184065"
   ```

8. 修改配置文件 {tidb_ansible_path}/conf/drainer_file_drainer.toml (注意下游不同的类型, 对配置文件的命名也不一样。可以参考 [drainer_mysql_drainer.toml 部分](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/deploy/#第-3-步部署-drainer))

   ```toml
   [syncer]
   # downstream storage, equal to --dest-db-type
   # Valid values are "mysql", "file", "tidb", "kafka".
   db-type = "file"
   
   # Uncomment this if you want to use "file" as "db-type".
   [syncer.to]
   # default data directory: "{{ deploy_dir }}/data.drainer"
   dir = "data.drainer"
   ```

9. 部署 Drainer

   ```bash
   shell> ansible-playbook deploy_drainer.yml
   ```

10. 启动 Drainer

   ```bash
   shell> ansible-playbook start_drainer.yml
   ```

11. 更新监控信息，如果新部署 TiDB binlog 会新增 Binlog 的 dashboard。
    
   ```bash
   shell> ansible-playbook rolling_update_monitor.yml --tags=prometheus
   ```

### 5.2 关闭 TiDB binlog

#### 当前集群拓扑信息

组件 | 机器 (IP端口) | 数量
---- | ---- | ----
TiDB | 172.16.4.243:4807 、 172.16.4.242:4807 | 2
PD | 172.16.4.243:12779 、172.16.4.242:12779 、172.16.4.240:12779 | 3
TiKV | 172.16.4.239:27173 、172.16.4.240:27173 、172.16.4.238:27173 | 2
Pump |172.16.4.242:8250 、172.16.4.241:8250 | 2
Drainer |172.16.4.239:8249 | 1

调整后集群目标拓扑信息

组件 | 机器 (IP端口) | 数量
---- | ---- | ----
TiDB | 172.16.4.243:4807 、 172.16.4.242:4807 | 2
PD | 172.16.4.243:12779 、172.16.4.242:12779 、172.16.4.240:12779 | 3
TiKV | 172.16.4.239:27173 、172.16.4.240:27173 、172.16.4.238:27173 | 2
~~Pump~~ |~~172.16.4.242:8250 、172.16.4.241:8250~~ |~~2~~
~~Drainer~~ |~~172.16.4.239:8249~~ | ~~1~~

#### 操作中的主要步骤

1. 确认 Drainer 的数量以及状态
   
   ```sql
   mysql> show drainer status;
   +--------------+-------------------+--------+--------------------+---------------------+
   | NodeID       | Address           | State  | Max_Commit_Ts      | Update_Time         |
   +--------------+-------------------+--------   +--------------------+---------------------+
   | node239:8249 | 172.16.4.239:8249 | online |    415012076096323585 | 2020-03-02 18:37:13 |
   +--------------+-------------------+--------   +--------------------+---------------------+
   ```


2. 下线所有消费的本集群的 Drainer。(注意：node_id 可以通过第一步获得 NodeID，也可以通过 ` binlogctl -cmd drainers ` 获得)
   
   ```bash
   shell>{tidb_ansible_path}/resources/bin/binlogctl -pd-urls=http://172.16.4.243:12779 -cmd offline-drainer -node-id "node239:8249" 
   ```

3. 编辑 inventory.ini 关闭 binlog ,  注释 Drainer 相关配置。

   ```toml
   ## Binlog Part
   [pump_servers]
   172.16.4.241
   172.16.4.242
   
   [drainer_servers]
   ## drainer_file ansible_host=172.16.4.239    initial_commit_ts="400962745252184065"
   
   ## Group variables
   [pd_servers:vars]
   # location_labels = ["zone","rack","host"]
   
   ## Global variables
   [all:vars]
   deploy_dir = /home/tidb/deploy_3.0.6
   
   ## Connection
   # ssh via normal user
   ansible_user = tidb
   
   cluster_name = test-cluster_3.0.6
   
   tidb_version = v3.0.6
   
   # process supervision, [systemd, supervise]
   process_supervision = systemd
   
   timezone = Asia/Shanghai
   
   enable_firewalld = False
   # check NTP service
   enable_ntpd = True
   set_hostname = False
   
   ## binlog trigger
   enable_binlog = False
   ```

4. 滚动重启 Tidb 关闭 binlog
   
   ```bash
   shell>ansible-playbook rolling_update.yml -t tidb
   ```

5. 确认 Pump 的数量以及状态
   
   ```sql
   mysql > show pump status;
   +------------+-------------------+--------   +--------------------+---------------------+
   | NodeID     | Address           | State  |    Max_Commit_Ts      | Update_Time         |
   +------------+-------------------+--------   +--------------------+---------------------+
   | node2:8250 | 172.16.4.242:8250 | online |    415056015094448129 | 2020-03-04 17:10:46 |
   | node1:8250 | 172.16.4.241:8250 | online |    415056014806089730 | 2020-03-04 17:10:45 |
   +------------+-------------------+--------   +--------------------+---------------------+
   ```

6. 手动下线所有的 Pump。(注意：node_id 可以通过第一步获得 NodeID，也可以通过 ` binlogctl -cmd drainers ` 获得)
   
   ```bash
   > {tidb_ansible_path}/resources/bin/binlogctl -pd-urls=http://172.16.4.243:12779 -cmd offline-pump -node-id "node2:8250" 
   > {tidb_ansible_path}/resources/bin/binlogctl -pd-urls=http://172.16.4.243:12779 -cmd offline-pump -node-id "node1:8250"  
   ```

7. 编辑 inventory.ini 关闭 binlog ,  注释 Pump 相关配置
   
   ```toml
   [pump_servers]
   #172.16.4.241
   #172.16.4.242
   
   [drainer_servers]
   
   ## Group variables
   [pd_servers:vars]
   # location_labels = ["zone","rack","host"]
   ```

## 六、操作后 Check 项

### 6.1 开启 TiDB binlog 

1. 可以通过 SQL 检查 Binlog 是否开启成功。（注意，也可以使用 binlogctl 来查询 Pump 和 Drainer 的状态）
   
   ```sql
   mysql> show global variables like 'log_bin';
   +---------------+-------+
   | Variable_name | Value |
   +---------------+-------+
   | log_bin       | 1     |
   +---------------+-------+
   1 row in set (0.01 sec)
   
   mysql> show pump status;
   +------------+-------------------+--------   +--------------------+---------------------+
   | NodeID     | Address           | State  |    Max_Commit_Ts      | Update_Time         |
   +------------+-------------------+--------   +--------------------+---------------------+
   | node1:8250 | 172.16.4.241:8250 | online |    415057221848137729 | 2020-03-04 18:27:31 |
   | node2:8250 | 172.16.4.242:8250 | online |    415057222136496130 | 2020-03-04 18:27:32 |
   +------------+-------------------+--------   +--------------------+---------------------+
   2 rows in set (0.01 sec)
   
   mysql> show drainer status;
   +--------------+-------------------+--------   +--------------------+---------------------+
   | NodeID       | Address           | State  |    Max_Commit_Ts      | Update_Time         |
   +--------------+-------------------+--------   +--------------------+---------------------+
   | node239:8249 | 172.16.4.239:8249 | online |    415057228441583618 | 2020-03-04 18:27:56 |
   +--------------+-------------------+--------   +--------------------+---------------------+
   1 row in set (0.01 sec)
   ```

2. 通过监控查看服务状态
   ![监控](media/binlog.png)


### 6.2 关闭 TiDB binlog

1. 可以通过 SQL 检查 Binlog 是否关闭成功。（注意，也可以使用 binlogctl 来查询 Pump 和 Drainer 的状态）
   
   ```sql
   mysql> show global variables like 'log_bin';
   +---------------+-------+
   | Variable_name | Value |
   +---------------+-------+
   | log_bin       | 0     |
   +---------------+-------+
   1 row in set (0.01 sec)
   
   mysql> show pump status;
   +------------+-------------------+---------   +--------------------+---------------------+
   | NodeID     | Address           | State   |    Max_Commit_Ts      | Update_Time         |
   +------------+-------------------+---------   +--------------------+---------------------+
   | node1:8250 | 172.16.4.241:8250 | offline |    415073332930019329 | 2020-03-05 11:31:48 |
   | node2:8250 | 172.16.4.242:8250 | offline |    415073371386544129 | 2020-03-05 11:34:16 |
   +------------+-------------------+---------   +--------------------+---------------------+
   2 rows in set (0.00 sec)
   
   mysql> show drainer status;
   +--------------+-------------------+---------   +--------------------+---------------------+
   | NodeID       | Address           | State   |    Max_Commit_Ts      | Update_Time         |
   +--------------+-------------------+---------   +--------------------+---------------------+
   | node239:8249 | 172.16.4.239:8249 | offline |    415072225463369729 | 2020-03-05 10:21:27 |
   +--------------+-------------------+---------   +--------------------+---------------------+
   1 row in set (0.02 sec)
   ```

## 七、常见问题

常见问题请参考 [FAQ](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/faq/) 以及 [常见错误修复](https://pingcap.com/docs-cn/stable/reference/tidb-binlog/troubleshoot/error-handling/)
- Q: 下线 Pump 或者 Drainer 能否使用 ansible-playbook stop.yml --tags pump 和 ansible-playbook stop_drainer.yml ？
- A: 不能，使用 ansibl-playbook stop.yml 后 pump 和 Drainer 还是处于 pause 的状态。并不是 offline 。下线需要使用 binlogctl -cmd offline-pump 和  binlogctl -cmd offline-drainer 

