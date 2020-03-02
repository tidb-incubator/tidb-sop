# 01-线上集群2.1升级3.0踩坑实践

## 一、背景 / 目的

分布式数据库集群运维过程有一定的复杂性和繁琐性，3.0 版本是目前被广泛使用的版本，相比 2.1 有大幅度增加性能，以及很多新增的功能和特性，整体架构、配置也有较大的优化。该篇根据广大用户的升级经验，尽可能将 Release-2.1 升级到 Release-3.0 的准备工作、升级过程中注意事项、升级后重点关注列举详细，做到防患于未然。为 Release-3.0 版本的优秀特性和产品性能在业务场景中广泛使用提供文档依托。

适用人群默认为熟悉 2.1 版本的使用，但是没有做过大版本升级的。

## 二、操作前的 Check 项

### 2.1 备份原集群修改过的 TiDB、TiKV 和 PD 参数

#### 创建临时目录，备份升级前的参数配置

```text
mkdir -p /tmp/tidb_update_3.0/conf
mkdir -p /tmp/tidb_update_3.0/group_vars
```

## 五、操作后 Check 监控项

登录 Grafana 页面 [http://{grafana-ip}:{grafana-port}](http://{grafana-ip}:{grafana-port}) 用户名和密码：inventory.ini 有配置 查看 overview 页面， Overview 页面的 Services Port Status 状态是否均为绿色的 up 状态； ![](../.gitbook/assets/15831579646252.jpg)

