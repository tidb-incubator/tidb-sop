# 04 现有集群 TiKV 如何从单实例部署调整为多实例部署
> 王贤净 2020 年 2 月 20 日

## 一、背景 / 目的
因业务增长以及其他不可控因素，需要对 TiKV 节点进行扩容操作。扩容方式包括添加新机器扩容 TiKV 节点 [使用 ansible 扩容](https://pingcap.com/docs-cn/stable/how-to/scale/with-ansible/) 或者对当前 TiKV 节点进行扩容（如当前节点挂载新的磁盘）等方式。该文档针对的场景为直接在当前节点进行扩容操作，即单机单实例部署调整为单机多实例部署。
## 二、相关概念及原理解释
单机多实例部署时，注意给 TiKV 节点打上 label 标签。PD 理解 TiKV 拓扑结构后，会根据提供的拓扑信息作出最优的调度。相关原理可参考 [集群拓扑信息详解](https://pingcap.com/docs-cn/stable/how-to/deploy/geographic-redundancy/location-awareness/)

## 三、操作前 Check 项
### 3.1 检查机器资源，确保单机可以部署两个 tikv 实例
- 每个实例能分配 16c，32G 内存
- 单个 tikv 硬盘大小配置建议 PCI-E SSD 不超过 2 TB，普通 SSD 不超过 1.5 TB，需要确认磁盘、IO 足够（可单盘或者多盘）

### 3.2 端口预留，检查端口是否被占用，当前是否可用等
```
shell> netstat -anp |grep 端口号（如 tikv_port、tikv_status_port）
```
### 3.3 检查集群当前的状态，是否正常，可通过监控或者 pd-ctl 查看。
- 监控面板：查看 cluster-overview → service port status 服务是否正常。
- pd-ctl：执行 health 显示集群健康信息

## 四、注意事项
 1. label 要和真实物理拓扑一致，否则标签无意义；
 2. 单机单实例部署调整为单机多实例部署时，同一台机器上注意区分 deploy_dir、端口、labels 标签；
    - 同一个机器上部署多个 tikv 节点时，新的 tikv 需要使用不同的 depoly_dir、端口等
 3. inventory.ini 配置文件中 ps_servers：vars 中的标签与 TiKV 中的标签要一致；
    - 设置标签后，使 PD 理解 tikv 集群的拓扑结构，从而进行最优调度
 4. 调低调度参数，防止 balance 过快。
 
 注意：使用 pd-ctl 执行 config show 命令可以查看所有的调度参数，执行 config set {key} {value} 可以调整对应参数的值，需调整参数如下：
    - leader-schedule-limit：控制 transfer leader 调度的并发数
    - region-schedule-limit：控制增删 peer 调度的并发数
    - replica-schedule-limit：控制同时进行 replica 调度的任务个数

## 五、操作步骤
### 5.1 集群拓扑信息
该扩容操作是将 238 机器上的 TiKV 节点由单机单实例状态调整为单机多实例部署。

#### 当前集群拓扑信息

组件 |机器 IP:端口 |  数量 
------- | ------- | ---
TiDB | 172.16.4.243:4807<br>172.16.4.242:4807 | 2 |
PD | 172.16.4.243:12779<br>172.16.4.242:12779<br>172.16.4.240:12779 | 3 |
TiKV | 172.16.4.239:27173<br>172.16.4.240:27173<br>172.16.4.238:27173  | 3 |

当前集群 label 的状态为空

```
./pd-ctl -i -u http://172.16.4.243:12779

» label
null

» config show replication
{
  "max-replicas": 3,
  "location-labels": "",
  "strictly-match-label": "false"
}
```

#### 调整后集群目标拓扑信息

组件 |机器 IP:端口 |  数量 
------- | ------- | ---
TiDB | 172.16.4.243:4807<br>172.16.4.242:4807 | 2 |
PD | 172.16.4.243:12779<br>172.16.4.242:12779<br>172.16.4.240:12779 | 3 |
TiKV | 172.16.4.239:27173<br>172.16.4.240:27173<br>**172.16.4.238:27173/labels="host=tikv1"**<br>**172.16.4.238:27174/labels="host=tikv1"**  | 4 |

- 这里以调整 1 个 tikv 举例，如果要调整所有的 tikv ，操作类似

### 5.2 操作中的关键步骤
单机单实例部署到单机多实例部署过程中，首先采用扩容方式为集群新加一个 tikv 节点，本次操作时，扩容时先对新节点打上 label 标签，扩容完成后，再使用 pd-ctl 在线对 238 节点上原有的 tikv 节点（即 tikv1-1）打上 label 标签。也可采用官网扩容方式直接添加 tikv 节点，添加成功后，在线对单机两个 tikv 节点进行打 label 操作 [详细命令参考官网链接](https://pingcap.com/docs-cn/stable/reference/tools/pd-control/#store-delete--label--weight-store_id--jqquery-string)

#### 0.首先将原 tikv 的内存配置减半
修改 tidb-ansible/conf/tikv.yml 中：

- capacity = MEM_TOTAL * 0.5 / TiKV 实例数量
- high-concurrency、normal-concurrency 和 low-concurrency 三个参数 为：TiKV 实例数量 * 参数值 = CPU 核心数量 * 0.8
- 需要重启该 tikv 实例

#### 1.修改 inventory.ini 文件，扩容一个 tikv 节点。
- 其中 ‘host’ 只是标签的名字，可以修改，但上下要统一
- 相同主机上的多个 tikv，label 要相同

注意：多实例场景需要额外配置 status 端口，**tikv_status_port** 指的是上报 TiKV 状态的通信端口。

```ini
# TiDB Cluster Part
[tidb_servers]
172.16.4.243 tidb_port=4807 tidb_status_port=11793
172.16.4.242 tidb_port=4807 tidb_status_port=11793

[tikv_servers]
172.16.4.239 tikv_port=27173 tikv_status_port=27193
172.16.4.240 tikv_port=27173 tikv_status_port=27193
tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"
tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv1"

... ...

## Group variables
[pd_servers:vars]
location_labels = ["host"]

## Global variables
[all:vars]
deploy_dir = /home/tidb/deploy_3.0.7
```

#### 2.初始化新增节点

```
ansible-playbook bootstrap.yml -l tikv1-2
```

#### 3.部署新增节点

```
ansible-playbook deploy.yml -l tikv1-2
```

#### 4.启动新增节点

```
ansible-playbook start.yml -l tikv1-2
```

注意：新节点启动成功后，已打上 label 标签，建议使用 pd-ctl 验证 label 的状态，确保已成功打上 label 标签。

```
./pd-ctl -i -u http://172.16.4.243:12779
» label
[
  {
    "key": "host",
    "value": "tikv1"
  }
]

» store 2409
{
  "store": {
    "id": 2409,
    "address": "172.16.4.238:27174",
    "labels": [
      {
        "key": "host",
        "value": "tikv1"
      }
    ],
    "version": "3.0.7",
    "state_name": "Up"
  },
  "status": {
… ...
  }
}
```
#### 5.更新 prometheus 监控

```
ansible-playbook rolling_update_monitor.yml --tags=prometheus
```

注意：更新监控后，新扩容的节点能够正常显示在监控中（tikv 节点由初始状态 3 个变成扩容后的 4 个节点）。
![监控](./media/sop2-1.png)


#### 6.将 238 机器上原有的 tikv 节点（即 tikv1-1，他的 store_id=4）打上 label 标签

```
» store label 4 host tikv1      # 给 tikv1-1 打 label
Success!

» store 4
{
  "store": {
    "id": 4,
    "address": "172.16.4.238:27173",
    "labels": [
      {
        "key": "host",
        "value": "tikv1"
      }
    ],
    "version": "3.0.7",
    "state_name": "Up"
  },
  "status": {
… ...
  }
}

» label
[
  {
    "key": "host",
    "value": "tikv1"
  },
  {
    "key": "host",
    "value": "tikv1"
  }
]
```

#### 7.设置 location-labels，该参数需要与 tikv 的 labels 名字对应，这样 PD 才能知道这些 labels 代表了 tikv 的拓扑结构。

注意：必须同时配置 PD 的 location-labels 和 tikv 的 labels 参数，否则 labels 不会生效。

```
» config set location-labels "host"
Success!

» config show replication
{
  "max-replicas": 3,
  "location-labels": "host",
  "strictly-match-label": "false"
}
```

## 六、操作后 Check 项

### 1.从监控中查看 TiKV 节点 label 的分布情况以及调度情况（查看 PD leader 的监控）
![label 监控](./media/sop2-2.png)

## 七、常见问题

### Q1.部署集群时，label 标签已设置，后面检查发现集群没有打 label 标签？
  A.inventory.ini 中未设置该内容，仍为注释状态
  
  ![inventory.ini](./media/sop2-3.png)


