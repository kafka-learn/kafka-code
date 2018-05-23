---

title: "Kafka源码结构"
date: 2018-5-22 10:00

---

[TOC]

# 目录结构

## 源码文件结构

目录 | 作用
--- | ---
bin | 存放可直接在Linux或Windows上运行的.sh文件和.bat文件，包含Kafka常用操作以及ZooKeeper便捷脚本
checkstyle | 存放代码规范检查文档
clients | 客户端的实现
config | 存放配置文件
connetct | Kafka Connect工具的实现
core | 核心模块
docs | 官方文档
examples | Kafka生产者消费者简单Demo
jmh-benchmarks | 基准测试模块
log4j-appender | 日志模块
streams | Kafka Streams客户端库
tools | 工具类

## 核心模块结构

目录 | 作用
--- | ---
admin | 管理模块，操作和管理topic， broker, consumer group， records等
api | 封装调用
client | Producer生产的元数据信息的传递
cluster | 存活的Broker集群、分区、副本以及他们的底层属性和相互关系
common | 异常类、枚举类、格式化类、配置类等
consumer | 旧版本的废弃消费者类
controller | Kafka集群控制中心的选举，分区状态管理，分区副本状态管理，监听ZooKeeper数据变化等
coordinator | GroupCoordinator处理一般组成员资格和偏移量。transaction管理事务
javaapi | 给java调用的生产者、消费者、消息集api
log | 管理log，它是消息存储的形式，可对应到磁盘上的一个文件夹
message | 由消息封装而成的一个压缩消息集
metrics | Kafka监控模块
network | 网络管理模块，对客户端连接的处理
producer | 旧版本的废弃生产者类
security | 权限管理
serializer | 消息序列化与反序列化处理
server | 服务器端的实现
tools | 各种控制台工具的实现
utils | 工具类
zk | 提供与ZooKeeper交互的管理方法和在管道之上的更高级别的Kafka特定操作
zookeeper | 一个促进管道传输请求的ZooKeeper客户端
