---
layout:     post
title:      apache kafka（入门教程轻松学）
subtitle:   Apache Kafka-核心组件和流程-副本管理器-设计-原理
date:       2019-08-23
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - 数据开发
    - kafka
    
---



#### 目录
>* [kafka简介](https://lianxiaobao.github.io/2019/08/23/Apache-kafka%E7%AE%80%E4%BB%8B/)
>* [kafka安装和使用](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka%E5%AE%89%E8%A3%85%E5%92%8C%E4%BD%BF%E7%94%A8/)
>* [kafka核心概念](https://lianxiaobao.github.io/2019/08/23/Apache-kafka%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5/)
>* [kafka核心组件和流程--控制器](https://lianxiaobao.github.io/2019/08/23/Apache-kafka%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5%E7%BB%84%E5%BB%BA%E5%92%8C%E6%B5%81%E7%A8%8B-%E6%8E%A7%E5%88%B6%E5%99%A8-%E8%AE%BE%E8%AE%A1-%E5%8E%9F%E7%90%86/)
>* [kafka核心组件和流程--协调器](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6%E5%92%8C%E6%B5%81%E7%A8%8B-%E5%8D%8F%E8%B0%83%E5%99%A8(%E6%B6%88%E8%B4%B9%E8%80%85%E5%92%8C%E7%BB%84%E5%8D%8F%E8%B0%83%E5%99%A8)-%E8%AE%BE%E8%AE%A1-%E5%8E%9F%E7%90%86/)
>* [kafka核心组件和流程--日志管理器](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka-%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6%E5%92%8C%E6%B5%81%E7%A8%8B-%E6%97%A5%E5%BF%97%E7%AE%A1%E7%90%86%E5%99%A8-%E8%AE%BE%E8%AE%A1-%E5%8E%9F%E7%90%86/)
>* [kafka核心组件和流程--副本管理器](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka-%E6%A0%B8%E5%BF%83%E7%BB%84%E4%BB%B6%E5%92%8C%E6%B5%81%E7%A8%8B-%E5%89%AF%E6%9C%AC%E7%AE%A1%E7%90%86%E5%99%A8-%E8%AE%BE%E8%AE%A1-%E5%8E%9F%E7%90%86/)
>* [kafka编程实战](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka-%E7%BC%96%E7%A8%8B%E5%AE%9E%E6%88%98-java%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%BC%80%E5%8F%91%E4%BE%8B%E5%AD%90/)
>* [apache kafka在zookeeper中存储结构](https://lianxiaobao.github.io/2019/08/19/kafka%E5%9C%A8zookeeper%E4%B8%AD%E7%9A%84%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84/)




本章简单介绍了副本管理器，副本管理器负责分区及其副本的管理。副本管理器具体的工作流程可以参考牟大恩所著的《Kafka入门与实践》。

##副本管理器
副本机制使得kafka整个集群中，只要有一个代理存活，就可以保证集群正常运行。这大大提高了Kafka的可靠性和稳定性。

Kafka中代理的存活，需要满足以下两个条件：

* 存活的节点要维持和zookeeper的session连接，通过zookeeper的心跳机制实现
* Follower副本要与leader副本保持同步，不能落后太多。


满足以上条件的节点在ISR中，一旦宕机，或者中断时间太长，Leader就会把同步副本从ISR中踢出。

所有节点中，leader节点负责接收客户端的读写操作，follower节点从leader复制数据。

副本管理器负责对副本管理。由于副本是分区的副本，所以对副本的管理体现在对分区的管理。

在第三章已经对分区和副本有了详细的讲解，这里再介绍两个重要的概念，LEO和HW。

* LEO是Log End Offset缩写。表示每个分区副本的最后一条消息的位置，也就是说每个副本都有LEO。
* HW是Hight Watermark缩写，他是一个分区所有副本中，最小的那个LEO。
看下图：

![](http://ww2.sinaimg.cn/large/006y8mN6ly1g692b7eto2j31bu0g0taa.jpg)

分区test-0有三个副本，每个副本的LEO就是自己最后一条消息的offset。可以看到最小的LEO是Replica2的，等于3，也就是说HW=3。这代表offset=4的消息还没有被所有副本复制，是无法被消费的。而offset<=3的数据已经被所有副本复制，是可以被消费的。

 

副本管理器所承担的职责如下：

* 副本过期检查
* 追加消息
* 拉取消息
* 副本同步过程
* 副本角色转换
* 关闭副本
再此就不再一一讲解了，详情可以参考牟大恩所著的《Kafka入门与实践》。

下一步：开始[《kafka编程实战》](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka-%E7%BC%96%E7%A8%8B%E5%AE%9E%E6%88%98-java%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%BC%80%E5%8F%91%E4%BE%8B%E5%AD%90/)的学习
