

# 前言

## 版本

版本 | 发表时间 | 发表人 | 主要修改内容
---- | ---- | ---- |----
V1.0 | 20200301 | PingCAP  UE team | 第一版，有错误修改请邮件给 support@pingcap.com

TiDB 作为一款新一代的分布式关系型数据库，在日常运维上会和传统的关系型数据库有一定的区别，用户生态团队给大量 TiDB 用户提供了社区技术支持，我们根据大家经常提问的一些运维问题，进行收敛，会逐步推出《TiDB 运维手册》，前期计划包括 SOP、POC、Case Stuady 三个系列。

本期是 SOP（`Standard Operation Procedure`） 系列，是第一个版本，一共包括 10 个主题，每个主题我们都会通过标准的 step by step 步骤，来完整实现对某一个常见运维目标的操作。希望对大家有用，如果在操作过程遇到什么问题、或者你有什么需求，可以通过邮件、或者 AskTUG 进行讨论与反馈（ https://asktug.com/t/topic/33145 ），我们会定期 review 、修正内容、增加新的主题，进行版本迭代。

> V1.0 目录:

    SOP 之 -- 线上集群升级
    SOP 之 -- 线上集群扩缩容
    SOP 之 -- 现有集群 TiKV 如何从单实例部署调整为多实例部署
    SOP 之 -- 现有集群如何从单机房调整为跨机房部署
    SOP 之 -- 关机临时维护某线上主机
    SOP 之 -- 在线表结构变更（Online DDL）
    SOP 之 -- 线上集群开启 / 关闭 Binlog 
    SOP 之 -- Prometheus 监控迁移流程
    SOP 之 -- 如何修改 TiDB/TiKV/PD IP
    SOP 之 -- 如何修改 TiDB/TiKV/PD 端口



 
 
 
 
 

 
 
 
 
 
 
 
 
 
 
