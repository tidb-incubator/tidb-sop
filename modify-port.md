# 10 现网如何修改 TiDB、TiKV、PD 的 port 
> 荣毅龙  2020 年 3 月 10 日

## 一、背景 / 目的
        因为机房网络调整，需要更改 TiDB、TiKV、PD 实例端口信息
## 二、相关概念及原理解释
        修改端口信息前，需要和网络规划保持一致，并且避免端口冲突的问题
## 三、操作前 Check 项
1. 端口预留，检查端口是否被占用，当前是否可用等

    ```
        netstat -anp |grep 端口号（如 tikv_port、tikv_status_port）
    ```
2. 检查集群当前的状态，是否正常，可通过监控或者 `pd-ctl` 查看
    监控面板：查看 `cluster-overview → service port status` 服务是否正常
    pd-ctl：执行 `health` 显示集群健康信息

## 四、注意事项
1. tidb和tikv调整端口时，需要重启，会影响正在运行的业务，最好可以在业务停止或者低峰期进行操作
2. 调整端口会也影响到告警信息，如果有其他告警对接，可能也需要调整配置

## 五、集群拓扑信息
组件 |机器 IP:端口 | 目标机器 IP:端口 | 数量 
---- | ---- | ---- | ----
TiDB | 172.16.4.242 tidb_port=4807 tidb_status_port=11793<br>172.16.4.243 tidb_port=4807 tidb_status_port=11793 | 172.16.4.242 tidb_port=4807 tidb_status_port=11793<br>172.16.4.243 tidb_port=4817 tidb_status_port=12793| 2 |
PD | 172.16.4.243 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.242 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.240 pd_client_port=12779 pd_peer_port=2787 |172.16.4.243 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.242 pd_client_port=12779 pd_peer_port=2787<br>172.16.4.240 pd_client_port=12779 pd_peer_port=2787 | 3 |
TiKV | 172.16.4.239 tikv_port=27173 tikv_status_port=27193<br>172.16.4.240 tikv_port=27173 tikv_status_port=27193<br>tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"<br>tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2" | 172.16.4.239 tikv_port=27173 tikv_status_port=27193<br>172.16.4.240 tikv_port=27273 tikv_status_port=27293<br>tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"<br>tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2" | 3 |



## 六、操作中的关键步骤
### 6.1 修改 tidb 实例端口 tidb_port 和 tidb_status_port
> 以 172.16.4.243 服务器 tidb_port 端口从 4807 修改为 4817，tidb_status_port 端口从 11793 修改为 12793 为例

1. 在ansible中控机上使用tidb用户，停止需要修改的服务器上的tidb-server服务

    ```
        su - tidb
        cd <ansible_deploy>
        ansible-playbook stop.yml -l 172.16.4.243 -t tidb
    ```        

2. 在中控机上修改 inventory.ini 文件
   
    ```
        # TiDB Cluster Part
        [tidb_servers]
        172.16.4.243 tidb_port=4817 tidb_status_port=12793
        172.16.4.242 tidb_port=4807 tidb_status_port=11793
    ```
3. 在tidb服务器 172.16.4.243 上，使用root用户修改/etc/systemd/system/目录下文件名tidb-4807.service 为 tidb-4817.service
    
    ```
        cd /etc/systemd/system/
        mv tidb-4807.service tidb-4817.service
    ```

4. 在tidb服务器 172.16.4.243 上，使用 tidb 用户修改 <deploy_dir>/scripts 目录下文件run_tidb.sh , start_tidb.sh , stop_tidb.sh 将原端口修改为目标端口
    
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
            -P 4817 \
            --status="12793" \
            --advertise-address="172.16.4.243" \
            --path="172.16.4.243:12779,172.16.4.242:12779,172.16.4.240:12779" \
            --config=conf/tidb.toml \
            --log-slow-query="/home/tidb/deploy_3.0.7/log/tidb_slow_query.log" \
            --log-file="/home/tidb/deploy_3.0.7/log/tidb.log" 2>> "/home/tidb/deploy_3.0.7/log/tidb_stderr.log"
        ---------------------------

        vi start_tidb.sh
        ---------------------------
        #!/bin/bash
        set -e

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        sudo systemctl start tidb-4817.service
        ---------------------------

        vi stop_tidb.sh 
        ---------------------------
        #!/bin/bash
        set -e

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        sudo systemctl stop tidb-4817.service
        ---------------------------
    ```

5. 在中控机上，使用 tidb 用户启动 tidb 服务

    ```
        ansible-playbook start.yml -l 172.16.4.243 -t tidb
    ```

6. 在中控机上，使用 tidb 用户更新 prometheus 监控
    
    ```
        ansible-playbook rolling_update_monitor.yml --tags=prometheus
    ```
        
7. 更新后查看监控项是否有显示新的端口信息,可以查看 tidb--->server---->uptime 监控信息

### 6.2 修改 tikv 实例端口
> 以 172.16.4.240 服务器 tikv_port 端口从 27173 修改为 27273，tikv_status_port 端口从 27193 修改为 27293
        
1. 在中控机，使用 tidb 用户修改参数max-store-down-time时间，默认为30min，修改为足够长时间 （举例修改为60min），或者完全关闭集群

    ```
        su - tidb
        cd <deploy_dir>/resource/bin
        ./pd-ctl -u http://172.16.4.240:12779 -i
        >config set max-store-down-time 60m
    ```


    解释：max-store-down-time 为 PD 认为失联 store 无法恢复的时间，当超过指定的时间没有收到 store 的心跳后，PD 会在其他节点补充副本

     
2. 在中控机，使用 tidb 用户关闭 172.16.4.240 tikv 实例
   
    ```
        su - tidb
        cd <deploy_dir>
        ansible-playbook stop.yml  -l 172.16.4.240 -t tikv
    ```        
        
3. 在中控机，使用tidb用户修改inventory.ini中的 tikv 端口

    ```
        su - tidb
        cd <deploy_dir>
        vi inventory.ini
        [tikv_servers]
        172.16.4.239 tikv_port=27173 tikv_status_port=27193
        172.16.4.240 tikv_port=27273 tikv_status_port=27293
        tikv1-1 ansible_host=172.16.4.238 tikv_port=27173 tikv_status_port=27193 labels="host=tikv1"
        tikv1-2 ansible_host=172.16.4.238 tikv_port=27174 tikv_status_port=27194 deploy_dir=/home/tidb/deploy2_3.0.7 labels="host=tikv2"
    ```
    
4. 在 tikv 服务器 172.16.4.240 上，使用root用户修改/etc/systemd/system/目录下文件名tikv-27173.service 为 tikv-27273.service
    
    ```
        cd /etc/systemd/system/
        mv tikv-27173.service tikv-27273.service
    ```    
        
5. 在 tikv 服务器 172.16.4.240 上，使用 tidb 用户修改 <deploy_dir>/scripts 目录下文件run_tikv.sh , start_tikv.sh , stop_tikv.sh 将原端口修改为目标端口

    ```
        su - tidb
        cd <deploy_dir>/scripts
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
            --addr "0.0.0.0:27273" \
            --advertise-addr "172.16.4.240:27273" \
            --status-addr "172.16.4.240:27293" \
            --pd "172.16.4.243:12779,172.16.4.242:12779,172.16.4.240:12779" \
            --data-dir "/home/tidb/deploy_3.0.7/data" \
            --config conf/tikv.toml \
            --log-file "/home/tidb/deploy_3.0.7/log/tikv.log" 2>> "/home/tidb/deploy_3.0.7/log/tikv_stderr.log"
        ---------------------------

        vi start_tikv.sh
        ---------------------------
        #!/bin/bash
        set -e

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        sudo systemctl start tikv-27273.service
        ---------------------------


        vi stop_tikv.sh 
        ---------------------------
        #!/bin/bash
        set -e

        # WARNING: This file was auto-generated. Do not edit!
        #          All your edit might be overwritten!
        sudo systemctl stop tikv-27273.service
        ---------------------------

    ```

6. 在中控机上，使用 tidb 用户启动 tikv 服务

    ```
        ansible-playbook start.yml -l 172.16.4.240 -t tikv
    ```

7. 在中控机上，使用 tidb 用户更新 prometheus 监控

    ```
        ansible-playbook rolling_update_monitor.yml --tags=prometheus
    ```

8. 更新后查看监控项是否有显示新的端口信息,可以查看 detail-tikv--->cluster---->cpu 监控信息

9. 在中控机，使用 tidb 用户修改参数max-store-down-time时间，默认为30min，修改为原配置

    ```
        su - tidb
        cd <deploy_dir>/resource/bin
        ./pd-ctl -u http://172.16.4.240:12779 -i
        >config set max-store-down-time 30m

        解释：max-store-down-time 为 PD 认为失联 store 无法恢复的时间，当超过指定的时间没有收到 store 的心跳后，PD 会在其他节点补充副本
    ```

### 6.3 PD 修改端口
        PD 不建议直接修改端口，请参考扩容缩容章节，先进行扩容，再缩容

## 七、常见问题

    Q1. 端口已经被占用，报错无法启动，需要提前检查端口信息
        netstat -ntlp | grep 2779

    Q2. 修改端口信息只修改了配置文件，没有修改/etc/systemd/system下的service文件


