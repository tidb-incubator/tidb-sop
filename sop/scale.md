# 08 线上集群扩缩容操作及要点
> 沈刚 2020 年 2 月 15 日

## 一、背景

随着业务和数据量的增长，很容易遇到 TiDB 集群出现性能下降或存储空间不足的问题，对于分布式的 TiDB 来讲，可以通过扩容集群节点来分担负载、提高存储能力。在运维工作中还可能遇到服务器维修、更换等需求，在 TiDB 集群中也可以通过扩容新服务器，缩容旧服务器的方式做到应用无感知的日常维护操作。本文主要讲解 TiDB 集群在扩容缩容 TiDB 、 TiKV 、 PD 组件方面的准备工作、注意事项、操作步骤以及检查回退方案。

## 二、相关概念及原理解释

### 2.1 TiDB Server

TiDB Server 负责接收 SQL 请求，处理 SQL 相关的逻辑，并通过 PD 找到存储计算所需数据的 TiKV 地址，与 TiKV 交互获取数据，最终返回结果。TiDB Server 是无状态的，其本身并不存储数据，只负责计算，可以无限水平扩展，可以通过负载均衡组件（如LVS、HAProxy 或 F5）对外提供统一的接入地址。

### 2.2 PD Server

Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个：一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID。
PD 通过 Raft 协议保证数据的安全性。Raft 的 leader server 负责处理所有操作，其余的 PD server 仅用于保证高可用。建议部署奇数个 PD 节点。


### 2.3 TiKV Server

Placement Driver (简称 PD) 是整个集群的管理模块，其主要工作有三个：一是存储集群的元信息（某个 Key 存储在哪个 TiKV 节点）；二是对 TiKV 集群进行调度和负载均衡（如数据的迁移、Raft group leader 的迁移等）；三是分配全局唯一且递增的事务 ID。
PD 通过 Raft 协议保证数据的安全性。Raft 的 leader server 负责处理所有操作，其余的 PD server 仅用于保证高可用。建议部署奇数个 PD 节点。


## 三、操作前 check 项

1. 检查当前集群状态，各个组件正常运行
2. 检查扩容组件需要使用的端口是否被占用(参考命令：netstat -ntlp | grep {PORT})

 * TiDB 组件默认使用 `tidb_port=4000,tidb_status_port=10080`
 * TiKV 组件默认使用 `tikv_port_20160,tikv_status_port=20180`
 * PD 组件默认使用 `pd_client_port=2379,pd_peer_port=2380`

3. 如果扩容节点是全新的服务器，扩容之前确认扩容节点防火墙是否关闭，与原集群端口访问正常，否则扩容过程中可能会因为通讯不成功导致扩容失败。
4. 扩容之前确认 swap 永久关闭，避免服务器重启会自动开启 swap

```
关闭 swap 分区
$ swapoff -a

将 /etc/fstab 中 swap 相关设置行注释或者删除，禁用 swap 分区防止服务器重启开启 swap
```

5. 如果扩容节点是全新的机器，需要参考官方文档：[使用 TiDB Ansible 部署 TiDB 集群](https://pingcap.com/docs-cn/dev/how-to/deploy/orchestrated/ansible/)，检查下述步骤是否已完成

 * 配置新扩容节点与中控机的 SSH 免密
 * 在部署目标机器上安装 NTP 服务
 * 在部署目标机器上配置 CPUfreq 调节器模式
 * 在部署目标机器上添加数据盘 ext4 文件系统挂载参数
 * 使用 `ansible-playbook bootstrap.yml -l {新机器ip或别名}` 对新机器进行内核参数调整


## 四、注意事项

1. 缩容节点时，需要注意被移除的节点是否有混合部署其他服务，避免使用 IP 方式控制服务启停时影响其他服务。
2. 在缩容 TiKV 过程中，当缩容节点处于 `Offline` 状态时，千万不要停止 TiKV 进程，需要等待缩容节点状态变为 `Tombstone` 后才可以停止 TiKV 进程。
3. 本文档示例操作均为单机单实例环境，`inventory.ini` 配置文件按照 IP 方式填写，如果有单机多实例的扩缩容需求，可以按照官方文档单机多实例配置方式填写 `inventory.ini`，ansible-playbook 操作时使用 -l 加别名的方式替代 -l 加 IP 方式指定特定的组件服务。
4. 缩容 TiKV 节点时需要确认 TiKV 有没有设置 label，如果设置了 label，缩容 TiKV 节点后能否满足 region 副本的调度，具体可以参考官方文档：集群拓扑信息配置。如果 region 副本无法调度，会导致节点一直处于 `Offline` 状态，无法正常下线。
5. 在扩缩容过程中，因为会产生 region 以及 leader 的迁移调度，想要减小迁移调度对集群产生的影响，可以考虑调整 PD 调度参数 `region-schedule-limit` 以及 `leader-schedule-limit` 来控制调度速度，这两个参数默认分别为 4 和 4。
6. 多机房部署的扩缩容操作时，建议使用同等比例机器数量以及配置。



## 五、操作步骤
假设集群拓扑结构如下所示：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node1 | 172.16.10.1 | PD1 |
| node2 | 172.16.10.2 | PD2 |
| node3 | 172.16.10.3 | PD3,Monitor |
| node4 | 172.16.10.4 | TiDB1 |
| node5 | 172.16.10.5 | TiDB2 |
| node6 | 172.16.10.6 | TiKV1 |
| node7 | 172.16.10.7 | TiKV2 |
| node8 | 172.16.10.8 | TiKV3 |
| node9 | 172.16.10.9 | TiKV4 |

### 5.1 TiDB/TiKV 扩容操作步骤
假设，需要添加两个 TiDB 节点：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node101 | 172.16.10.101 | TiDB3 |
| node102 | 172.16.10.102 | TiDB4 |

1. 编辑 `inventory.ini` 文件，添加需要扩容的节点信息
在 `[tidb_servers]` 主机组最后添加要扩容的节点信息

```
[tidb_servers]
172.16.10.4
172.16.10.5
172.16.10.101
172.16.10.102
```

2. 初始化新增节点

```
$ ansible-playbook bootstrap.yml -l 172.16.10.101,172.16.10.102
```

3. 部署新增节点

```
$ ansible-playbook deploy.yml -l 172.16.10.101,172.16.10.102
```

4. 启动新节点服务

```
$ ansible-playbook start.yml -l 172.16.10.101,172.16.10.102
```

5. 按照 5.6 中更新监控的步骤更新 Prometheus 配置并重启
6. 访问监控平台，检查整个集群和新增节点的状态是否正常

**扩容 TiKV 组件的步骤与扩容 TiDB 组件步骤相同。**

### 5.2 PD 扩容操作步骤
假设，需要添加一个 PD 节点：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node103 | 172.16.10.103 | PD4 |

1. 编辑 `inventory.ini` 文件，添加需要扩容的节点信息
在 `[pd_servers]` 主机组最后添加要扩容的节点信息

```
[pd_servers]
172.16.10.1
172.16.10.2
172.16.10.3
172.16.10.103
```

2. 初始化新增节点

```
$ ansible-playbook bootstrap.yml -l 172.16.10.103
```

3. 部署新增节点

```
$ ansible-playbook deploy.yml -l 172.16.10.103
```

4. 登录新增的 PD 节点，编辑启动脚本：`{deploy_dir}/scripts/run_pd.sh`

```
* 删除 --initial-cluster=”xxxx”\ 配置行，注意不能通过注释方式，必须删除
* 在删除行的位置添加 --join=”http://172.16.10.1:2379” \，IP 地址可以是集群内现有 PD IP 地址中的任意一个，注意不要漏掉行尾的反斜杠\
* 在新增 PD 节点中手动启动 PD 服务
${deploy_dir}/scripts/start_pd.sh
```

> 注意：启动前需要通过 pd-ctl 工具执行 member 命令 确认当前 PD 节点的 health 状态均为 true ，否则 PD 服务会启动失败，同时日志中报错 [“join meet error”][error=”etcdserver:unhealthy cluster”]。

5. 使用 pd-ctl 工具检查新节点是否添加成功

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member
```

6. 更新集群配置

```
$ ansible-playbook deploy.yml
```

> 注意：TiKV 中的 PD Client 会缓存 PD 节点列表，但目前不会定期自动更新，只有在 PD leader 发生切换或 TiKV 重启加载最新配置后才会更新；为避免 TiKV 缓存的 PD 节点列表过旧的风险，在扩缩容 PD 完成后，PD 集群中至少要包含两个扩缩容操作前就已经存在的 PD 节点成员，如果不满足该条件需要手动执行 PD transfer leader 操作，更新 TiKV 中的 PD 缓存列表。PD transfer leader 的操作可以参考 4.7 节 ”PD transfer leader 操作步骤“。

7. 启动监控服务

```
$ ansible-playbook start.yml -l 172.16.10.103
```

8. 按照 5.6 中更新监控的步骤更新 Prometheus 配置并重启
9. 访问监控平台，检查整个集群和新增节点的状态是否正常

### 5.3 TiDB 缩容操作步骤
假设，要移除一个 TiDB节点：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node105 | 172.16.10.5 | TiDB2 |

1. 通过 Ansible 停止 node5 上的服务（包括 TiDB 组件服务和监控服务）

```
$ ansible-playbook stop.yml -l 172.16.10.5
```

2. 编辑 `inventory.ini` 文件，在 `[tidb_servers]` 和 `[monitored_servers]` 主机组中将对应的配置行注释

```
[tidb_servers]
172.16.10.4
#172.16.10.5  # 注释被移除节点

······

[monitored_servers]
172.16.10.4
#172.16.10.5  # 注释被移除节点
172.16.10.1
172.16.10.2
172.16.10.3
172.16.10.6
172.16.10.7
172.16.10.8
172.16.10.9
```

3. 按照 5.6 中更新监控的步骤更新 Prometheus 配置并重启
4. 访问监控平台，检查整个集群状态是否正常

### 5.4 TiKV 缩容操作步骤
假设，要移除一个 TiKV 节点：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node9 | 172.16.10.9 | TiKV4 |


1. 使用 pd-ctl 将 TiKV4 从集群中移除

```
* 查看要移除节点 TiKV4 的 store id
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d store
* 指定 store id 将 TiKV4 从集群中移除（TiKV4 的 store id 为10）
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d store delete 10
```

2. 通过监控平台或者 pd-ctl 判断节点是否下线成功（等待下线节点状态变为 `Tombstone` 就表示下线成功了）

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d store 10
```

3. 等待节点下线成功后，停止 node9 上的服务（包括 TiKV 组件服务和监控服务）

```
$ ansible-playbook stop.yml -l 172.16.10.9
```

4. 编辑 `inventory.ini` 文件，在 `[tikv_servers]` 和 `[monitored_servers]` 主机组中将对应的配置行注释

```
[tidb_servers]
172.16.10.4
172.16.10.5

······

[tikv_servers]
172.16.10.6
172.16.10.7
172.16.10.8
#172.16.10.9  # 注释被移除节点

[monitored_servers]
172.16.10.4
172.16.10.5
172.16.10.1
172.16.10.2
172.16.10.3
172.16.10.6
172.16.10.7
172.16.10.8
#172.16.10.9  # 注释被移除节点
```

5. 按照 5.6 中更新监控的步骤更新 Prometheus 配置并重启
6. 访问监控平台，检查整个集群状态是否正常

### 5.5 PD 缩容操作步骤
假设，要移除一个 PD 节点：

| Name | Host IP | Service |
| ---- | ------- | ------- |
| node2 | 172.16.10.2 | PD2 |


1. 使用 pd-ctl 将 PD2 从集群中移除

```
* 查看要移除节点 TiKV4 的 name
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member
* 指定 name 将 PD2 从集群中移除（PD2 的 name 为 pd2）
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member delete name pd2
```

2. 通过监控平台或者 pd-ctl 判断节点是否下线成功（等待结果没有 PD2 节点信息即为下线成功）

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member
```

3. 等待节点下线成功后，停止 node2 上的服务（包括 PD 组件服务和监控服务）

```
$ ansible-playbook stop.yml -l 172.16.10.2
```

4. 编辑 `inventory.ini` 文件，在 `[pd_servers]` 和 `[monitored_servers]` 主机组中将对应的配置行注释

```
[tidb_servers]
172.16.10.4
172.16.10.5

[pd_servers]
172.16.10.1
#172.16.10.2  # 注释被移除节点
172.16.10.3

······

[monitored_servers]
172.16.10.4
172.16.10.5
172.16.10.1
#172.16.10.2  # 注释被移除节点
172.16.10.3
172.16.10.6
172.16.10.7
172.16.10.8
172.16.10.9
```

5. 滚动升级整个集群

```
$ ansible-playbook deploy.yml
```

> 注意：TiKV 中的 PD Client 会缓存 PD 节点列表，但目前不会定期自动更新，只有在 PD leader 发生切换或 TiKV 重启加载最新配置后才会更新；为避免 TiKV 缓存的 PD 节点列表过旧的风险，在扩缩容 PD 完成后，PD 集群中至少要包含两个扩缩容操作前就已经存在的 PD 节点成员，如果不满足该条件需要手动执行 PD transfer leader 操作，更新 TiKV 中的 PD 缓存列表。PD transfer leader 的操作可以参考 4.7 节 ”PD transfer leader 操作步骤“。

6. 按照 5.6 中更新监控的步骤更新 Prometheus 配置并重启
7. 访问监控平台，检查整个集群状态是否正常

### 5.6 更新监控
不论是扩容还是缩容操作在最后都是需要更新监控的，需要更新 Prometheus 配置通知 Prometheus 有哪些新增服务或者哪些服务已经被去掉了，保证监控数据的正确性以及完整性。

```
$ ansible-playbook rolling_update_monitor.yml --tags=prometheus
```

更新完监控之后，可以通过浏览器访问监控平台：`http://{grafana_server_ip}:3000`，检查整个集群和新增节点的状态。

### 5.7 PD transer leader 操作步骤
* 查看当前 PD leader

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member leader show 
```

* 将 PD leader 从当前成员迁走，迁到哪个成员随机

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member leader resign
```

* 如果想要 PD leader 迁至指定成员可以使用 `member leader transfer {pd_name}` 命令

```
$ /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d member leader transfer {pd_name}
```

## 六、回退方案
### 6.1 扩容操作回退
如果在扩容节点服务没有加入集群之前，需要回退扩容操作，只需要按照缩容的步骤将服务停止并清理对应服务的部署文件，将 `inventory.ini` 中将回退节点注释即可。

### 6.2 TiDB 缩容操作回退
TiDB 组件缩容过程中如果有异常或者想取消缩容，可以按照扩容步骤重新操作即可。 

### 6.3 TiKV 缩容操作回退
#### 6.3.1 TiKV 节点下线处于 offline 状态，如何重新上线 TiKV 节点
可以通过 pd api 接口修改 Store 的状态：

```
* 通过 pd-ctl 工具查看需要重新上线的 Store id
$  /home/tidb/tidb-ansible/resources/bin/pd-ctl -u "http://172.16.10.1:2379" -d store
* 通过 pd api 接口修改 Store 状态为 Up
$ curl -X POST http://127.0.0.1:2379/pd/api/v1/store/<store-id>/state?state=Up
```

#### 6.3.2 TiKV 节点已经下线完成，如何重新上线 TiKV 节点

如果 TiKV 节点已经下线完成，想要重新将该 TiKV 节点上线，可以删除该节点上 {deploy_dir}/data 以及 {deploy_dir}/log/tikv* 相关历史文件后，参照扩容 TiKV 步骤进行扩容，但是需要注意：

* 如果是正常下线完成，等待 TiKV 节点状态变为 Tombstone 后，可以直接参照扩容 TiKV 步骤进行扩容
* 如果未正常下线，需要修改 tikv_port 端口，与原先该节点上的 tikv_port 不一致，因为 tikv 节点的信息会以 IP + tikv_port 作为唯一标识记录在 PD 中，目前该信息无法从 PD 中清除，所以需要更换 tikv_port 后重新上线。

### 6.4 PD 缩容操作回退

PD 组件缩容过程中如果有异常或者想取消缩容，可以按照扩容步骤重新操作即可，但在执行扩容操作之前需要删除 `{deploy_dir}/data.pd` 以及 `{deploy_dir}/log/pd*` 相关历史文件再进行扩容步骤。

## 七、操作后 Check 项
1. 扩缩容操作完成后，检查集群状态

 * 通过监控 Overview 面板检查所有节点状态都是 health 的
 * 通过 pd-ctl 查看 member 节点状态为 health，且有 leader 节点
 * 通过 pd-ctl 查看 store 节点状态为 health，各个 store 上 region 以及 leader 数量分布是否均匀

2. 缩容完成后检查空间，确认缩容操作无误，且集群正常后，可以将对应组件的数据文件以及日志文件进行清理

## 八、相关案例

AskTUG 相关问题：

* [https://asktug.com/t/tikv/980](https://asktug.com/t/tikv/980)
* [https://asktug.com/t/pd/965/2](https://asktug.com/t/pd/965/2)
* [https://asktug.com/t/k8s-tikv-tikv-running/1584](https://asktug.com/t/k8s-tikv-tikv-running/1584)
* [https://asktug.com/t/topic/2749/14](https://asktug.com/t/topic/2749/14)



