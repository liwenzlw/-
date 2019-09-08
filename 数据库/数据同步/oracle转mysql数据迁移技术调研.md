# 阿里数据迁移产品

## yugong(愚公)：

阿里巴巴去Oracle数据迁移同步工具(全量+增量,目标支持MySQL/DRDS)

原理：基于物化视图日志实现全量、增量同步

缺点：

* 不支持DDL,表结构发生变化，yugong会抛异常并停止java进程
* 只支持源数据库为ORACLE


## DataX

DataX本身作为数据同步框架，将不同数据源的同步抽象为从源头数据源读取数据的Reader插件，以及向目标端写入数据的Writer插件，理论上DataX框架可以支持任意数据源类型的数据同步工作。同时DataX插件体系作为一套生态系统, 每接入一套新数据源该新加入的数据源即可实现和现有的数据源互通。

## canal 

作为 MySQL binlog 增量获取和解析工具，可将变更记录投递到 MQ 系统中，比如 Kafka/RocketMQ，可以借助于 MQ 的多语言能力

## otter

名称：otter ['ɒtə(r)]

译意： 水獭，数据搬运工

语言： 纯java开发

定位： 基于数据库增量日志解析，准实时同步到本机房或异地机房的mysql/oracle数据库. 一个分布式数据库同步系统

otter已在阿里云推出商业化版本 数据传输服务DTS， 开通即用，免去部署维护的昂贵使用成本。

### 项目背景：
 阿里巴巴B2B公司，因为业务的特性，卖家主要集中在国内，买家主要集中在国外，所以衍生出了杭州和美国异地机房的需求，同时为了提升用户体验，整个机房的架构为双A，两边均可写，由此诞生了otter这样一个产品。

原理描述：

1. 基于Canal开源产品，获取数据库增量日志数据。 什么是Canal, 请点击

2. 典型管理系统架构，manager(web管理)+node(工作节点)

    a. manager运行时推送同步配置到node节点

    b. node节点将同步状态反馈到manager上

3. 基于zookeeper，解决分布式状态调度的，允许多node节点之间协同工作.

参考博客：http://blog.chuangzhi8.cn/posts/%E5%BC%82%E6%9E%84%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%81%E7%A7%BB%E4%B8%8E%E5%90%8C%E6%AD%A5-%E4%BA%8C-%E4%B9%8Botter.html


# 其他技术方案

## Oracle GoldenGate

有DDL和DML的限制、字段类型限制等