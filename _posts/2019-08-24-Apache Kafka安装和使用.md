---
layout:     post
title:      apache kafka（入门教程轻松学）
subtitle:   Apache Kafka安装和使用
date:       2019-08-23
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - 数据开发
    - kafka
    
---


本文转载于：https://blog.csdn.net/liyiming2017/article/details/82790040





# 第二章 Kafka安装和使用
 

## 单机环境

官方建议使用JDK 1.8版本，因此本文使用的环境都是JDK1.8。关于JDK的安装，本文不再详述，默认Java环境已经具备。

由于Kafka依赖zookeeper，kafka通过zookeeper现实分布式系统的协调，所以我们需要先安装zookeeper。

接下来我们按照如下步骤，一步步来安装kafka：

#### 1、下载zookeeper，解压。

下载地址：<https://zookeeper.apache.org/releases.html#download>

#### 2、创建zookeeper配置文件

在zookeeper解压后的目录下找到conf文件夹，进入后，复制文件zoo_sample.cfg，并命名为zoo.cfg

zoo.cfg中一共四个配置项，可以使用默认配置。

#### 3、启动zookeeper。

进入zookeeper根目录执行 bin/zkServer.sh start

![](http://ww4.sinaimg.cn/large/006y8mN6ly1g68z67yr60j30w405sdis.jpg)

#### 4、下载kafka，解压。

kafka 2.0版本下载地址：

<https://www.apache.org/dyn/closer.cgi?path=/kafka/2.0.0/kafka_2.11-2.0.0.tgz>

#### 5、修改kafka的配置文件

进入kafka根目录下的config文件夹下，打开[server.properties]() ,修改如下配置项

	zookeeper.connect=localhost:2181

	broker.id=0

	log.dirs=/tmp/kafka-logs

>* zookeeper.connect是zookeeper的链接信息，
>* broker.id是当前kafka实例的id，
>* log.dirs是kafka存储消息内容的路径。

#### 6、启动kafka

进入kafka根目录执行 [bin/kafka-server-start.sh config/server.properties]()

此命令告诉kaka启动时使用config/server.properties配置项



启动kafka后，如果控制台没有报错信息，那么kafka应该已经启动成功了，我们可以通过查看zookeeper中相关节点值来确认。步骤如下：

##### 1、启动zookeeper的client

进入zookeeper根目录下，执行 bin/zkCli.sh -server 127.0.0.1:2181。启动成功后如下图

![](http://ww1.sinaimg.cn/large/006y8mN6ly1g68zexs63rj319001sgmj.jpg)

##### 2、输入命令 ls /brokers，回车，可以看到如下信息：

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g68zg7dgezj30wo03ijt5.jpg)

这些子节点存储的就是kafka集群管理的数据。broker是kafka的一个服务单元实例

##### 3、我看看一下ids这个节点下的数据，输入命令 ls /brokers/ids，可以看到如下信息：

![](http://ww1.sinaimg.cn/large/006y8mN6ly1g68zgsr788j30wo02sgmg.jpg)

还记得我们在配置单机环境时，修改的kafka配置项broker.id=0 吗？这里的0就是表示那个kafka的实例已经加入了kafka集群。

 
## 集群环境
集群环境的搭建也很简单，在单机环境的基础上，让多个单机连接到同一个zookeeper即可。需要注意两点：

1、每个实例设置不同的broker.id。

2、如果多个实例部署在同一台服务器，还要注意修改log.dirs为不同目录，确保消息存储时不会有冲突。

集群环境的具体搭建，在此精简教程中不再做详细讨论。

 

## 发出你的第一条kafka消息
我们通过kafka带的工具来创建一个topic，然后尝试发送和消费一个消息，直观的去感受下kafka。

### 1、创建topic

进入kafka根目录，执行如下命令：

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic study

执行成功后，创建了study这个topic，如下图所示：
![](http://ww2.sinaimg.cn/large/006y8mN6ly1g68zmx8kmfj31bg022ab9.jpg)


此命令行有几个参数，分别指明了zookeeper的链接信息，分区和副本的数量等。关于分区和副本后续会仔细讲解，现在不用过多关注。

#### 2、启动消费者

我们开启一个消费者并且订阅study这个topic，执行如下命令:

	bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic study --from-beginning

看到如下图，光标停留在最前面，没有任何信息输出，说明启动消费者成功，此时在等待新的消息。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g68zn9acqvj31b801kdgp.jpg)

#### 3、开启生产者

新打开一个命令窗口，输入命令

	bin/kafka-console-producer.sh --broker-list localhost:9092 --topic study

启动成功后，如下图，等待你输入新的消息。
![](http://ww1.sinaimg.cn/large/006y8mN6ly1g68zozsh67j31ba02m75o.jpg)


#### 4、发送你的第一条消息

在上面生产者的窗口输入一条消息 hello kafka,点击回车，如下图：
![](http://ww3.sinaimg.cn/large/006y8mN6ly1g68zpv1la0j31be03ejsv.jpg)
此时切换到消费者的窗口，可以看到消费者已经消费到这条消息，在窗口中打印了出来。

![](http://ww3.sinaimg.cn/large/006y8mN6ly1g68zqa3x2oj31b205c40r.jpg)

至此我们走完了一个发送消息的流程，可以看到我们经历了创建topic、启动生产者、消费者、生产者生产消息、消费者消费消息，这几个步骤。

小结：通过本章节学习，相信你已经能够成功搭建起kafka单机环境，甚至集群环境。然后通过kafka自带的工具，直观的感受了kafka运转的整个过程。接下来的章节我们将会进入kafka的核心领域，也是本教程的重点章节，只有理解了kafka内在的设计理念和原理，才能做到活学活用。

下一步：开始[《kafka核心概念》]()的学习


