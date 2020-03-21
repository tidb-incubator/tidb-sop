# 09 现网如何修改 tidb，tikv，pd 的 ip 地址
> 荣毅龙  2020 年 3 月 10 日

## 一、背景 / 目的
    机房网络调整，需要更改 tidb，tikv，pd 实例 ip 地址信息.

## 二、相关概念及原理解释
    修改ip地址信息前，需要和网络规划保持一致，并且避免 ip 冲突的问题

## 三、操作前 Check 项
        检查集群当前的状态，是否正常，可通过监控或者 pd-ctl 查看。
        监控面板：查看 cluster-overview → service port status 服务是否正常。
        pd-ctl：执行 health 显示集群健康信息
## 四、注意事项
        1. tidb 和 tikv 调整ip地址时，需要重启，会影响正在运行的业务，最好可以在业务停止或者低峰期进行操作
        2. 调整端口会也影响到告警信息，如果有其他告警对接，可能也需要调整配置
        3. 以下调整为单独部署，如果和PD混合部署，不建议修改IP地址
        4. PD不建议修改IP，使用扩容缩容的方法
## 五、集群拓扑信息
组件 | 机器 IP:端口 | 目标机器 IP:端口 | 数量 
---- | ---- | ---- | ----
TiDB | 172.16.4.242 tidb_port=4807 tidb_status_port=11793<br>172.16.4.243 tidb_port=4807 tidb_status_port=11793 | 172.16.4.252 tidb_port=4807 tidb_status_port=11793<br>172.16.4.243 tidb_port=4807 tidb_status_port=11793| 2 |
PD | 172.16.4.243 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.242 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.240 pd_client_port=12779 pd_peer_port=2787 |172.16.4.243 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.242 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.240 pd_client_port=12779 pd_peer_port=2787 | 3 |
TiKV | 172.16.4.239 tikv_port=27173 tikv_status_port=27193<br>172.16.4.240 tikv_port=27173 tikv_status_port=27193<br>tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"<br>tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2" | 172.16.4.259 tikv_port=27173 tikv_status_port=27193<br>172.16.4.240 tikv_port=27173 tikv_status_port=27193<br>tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"<br>tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2" | 3 |



## 六、操作中的关键步骤
### 6.1 修改 tidb 实例IP地址
> 以 172.16.4.242 tidb实例为例,将172.16.4.242修改为172.16.4.252

1. 在ansible中控机上使用tidb用户，停止需要修改的服务器上的tidb-server服务

    ```
        su - tidb
        cd <ansible_deploy>
        ansible-playbook stop.yml -l 172.16.4.242 -t tidb
    ```   
     
2. 在中控机上修改 inventory.ini 文件

    ```
        # TiDB Cluster Part
        [tidb_servers]
        172.16.4.243 tidb_port=4807 tidb_status_port=11793
        172.16.4.252 tidb_port=4807 tidb_status_port=11793
    ```

3. 在tidb服务器 172.16.4.242 上，使用 tidb 用户修改 <deploy_dir>/scripts 目录下文件run_tidb.sh 将原 IP 地址修改为目标 IP 地址

    ```
        su - tidb
        cd <deploy_dir>/scripts
        vi run_tidb.sh
        ---------------------------
        #!/bin/bash
        set -e

        ulimit -n 1000000

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        DEPLOY_DIR=/home/tidb/deploy_3.0.7

        cd "${DEPLOY_DIR}" || exit 1

        export TZ=Asia/Shanghai

        exec bin/tidb-server \
            -P 4807 \
            --status="11793" \
            --advertise-address="172.16.4.252" \
            --path="172.16.4.243:12779,172.16.4.242:12779,172.16.4.240:12779" \
            --config=conf/tidb.toml \
            --log-slow-query="/home/tidb/deploy_3.0.7/log/tidb_slow_query.log" \
            --log-file="/home/tidb/deploy_3.0.7/log/tidb.log" 2>> "/home/tidb/deploy_3.0.7/log/tidb_stderr.log"
        ---------------------------
    ```
4. 修改ip地址，注意是否同网段，具体可以咨询网络工程师
 （1）查看ip地址配置的网卡信息

    ```
    [root@node239 ~]# ifconfig -a
      eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 172.16.4.239  netmask 255.255.254.0  broadcast 172.16.5.255
    ```

 （2）修改 `/etc/sysconfig/network-scripts/ifcfg-xxx` 对应的网卡信息，vi编辑修改 IP 信息
    重启网络 `service network restart`


5. 在中控机上，使用 tidb 用户启动 tidb 服务

    ```
        ansible-playbook start.yml -l 172.16.4.242 -t tidb
    ```
6. 在中控机上，使用 tidb 用户更新 prometheus 监控

    ```
        ansible-playbook rolling_update_monitor.yml --tags=prometheus
    ```
    
7. 更新后查看监控项是否有显示新的端口信息,可以查看 `tidb--->server---->uptime` 监控信息

### 6.2 修改 tikv 实例 IP 地址
> 以 172.16.4.239 服务器IP地址修改为 172.16.4.259

1. 在中控机，使用 tidb 用户修改参数 `max-store-down-time` 时间，默认为 30min，修改为足够长时间 （举例修改为 60min），或者完全关闭集群

    ```
        su - tidb
        cd <deploy_dir>/resource/bin
        ./pd-ctl -u http://172.16.4.240:12779 -i
        >config set max-store-down-time 60m

        解释：max-store-down-time 为 PD 认为失联 store 无法恢复的时间，当超过指定的时间没有收到 store 的心跳后，PD 会在其他节点补充副本
    ```

2. 关闭集群

    ```
        su - tidb
        cd <deploy_dir>
        ansible-playbook stop.yml
    ```        

3. 在中控机，使用tidb用户修改inventory.ini中的 tikv IP 地址

    ```
        su - tidb
        cd <deploy_dir>
        vi inventory.ini
        [tikv_servers]
        172.16.4.259 tikv_port=27173 tikv_status_port=27193
        172.16.4.240 tikv_port=27273 tikv_status_port=27293
        tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"
        tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2"

        其他监控信息配置也一并修改
    ```

4. 在 tikv 服务器 172.16.4.239 上，使用 tidb 用户修改 <deploy_dir>/scripts 目录下文件run_tikv.sh 将原 IP 地址修改为目标 IP 地址
   `su - tidb`
   `cd <deploy_dir>/scripts`
    
    ```
        vi run_tikv.sh
        ---------------------------
        #!/bin/bash
        set -e
        ulimit -n 1000000

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        cd "/home/tidb/deploy_3.0.7" || exit 1

        export RUST_BACKTRACE=1

        export TZ=${TZ:-/etc/localtime}

        echo -n 'sync ... '
        stat=$(time sync)
        echo ok
        echo $stat

        echo $$ > "status/tikv.pid"

        exec bin/tikv-server \
            --addr "0.0.0.0:27173" \
            --advertise-addr "172.16.4.259:27173" \
            --status-addr "172.16.4.259:27193" \
            --pd "172.16.4.243:12779,172.16.4.242:12779,172.16.4.240:12779" \
            --data-dir "/home/tidb/deploy_3.0.7/data" \
            --config conf/tikv.toml \
            --log-file "/home/tidb/deploy_3.0.7/log/tikv.log" 2>> "/home/tidb/deploy_3.0.7/log/tikv_stderr.log"
        ---------------------------
    ```        

5. 修改ip地址，注意是否同网段，具体可以咨询网络工程师
  (1) 查看ip地址配置的网卡信息
    [root@node239 ~]# ifconfig -a
      eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
      inet 172.16.4.239  netmask 255.255.254.0  broadcast 172.16.5.255

  (2) 修改/etc/sysconfig/network-scripts/ifcfg-xxx 对应的网卡信息，vi编辑修改 IP 信息
    重启网络 service network restart

6. 在中控机上，使用 tidb 用户启动 TIKV 服务

    ```
        ansible-playbook start.yml -l 172.16.4.259 -t tikv
    ```
7. 在中控机上，使用 tidb 用户更新 prometheus 监控

    ```
        ansible-playbook rolling_update_monitor.yml --tags=prometheus
    ```

8. 更新后查看监控项是否有显示新的端口信息,可以查看 `detail-tikv--->cluster---->cpu` 监控信息

9. 在中控机，使用 tidb 用户修改参数 `max-store-down-time` 时间，默认为 30min，修改为原配置

    ```
        su - tidb
        cd <deploy_dir>/resource/bin
        ./pd-ctl -u http://172.16.4.240:12779 -i
        >config set max-store-down-time 30m

        解释：max-store-down-time 为 PD 认为失联 store 无法恢复的时间，当超过指定的时间没有收到 store 的心跳后，PD 会在其他节点补充副本
    ```

### 6.3 PD 修改 IP 地址
PD 不建议直接修改 IP 地址，请参考扩容缩容章节，先进行扩容，再缩容

## 七、常见问题
Q1. IP 地址已经使用，提前在其他机器ping 新ip ，避免 IP重复使用

