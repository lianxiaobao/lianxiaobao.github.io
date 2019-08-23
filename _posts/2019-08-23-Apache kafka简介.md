---
layout:     post
title:      apache kafka（入门教程轻松学）
subtitle:   Apache Kafka简介
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

# 第一章 Kafka简介
### kafka的定位

提到kafka，不太熟悉或者稍有接触的开发人员，第一想法可能会觉得它是一个消息系统。其实Kafka的定位并不止于此。

Kafka官方文档介绍说，Apache Kafka是一个分布式流平台，并给出了如下解释：

流平台有三个关键的能力：

* 发布订阅记录流，和消息队列或者企业新消息系统类似。
* 以可容错、持久的方式保存记录流
* 当记录流产生时就进行处理


Kafka通常用于应用中的两种广播类型：

* 在系统和应用间建立实时的数据管道，能够可信赖的获取数据。
* 建立实时的流应用，可以处理或者响应数据流。


由此可见，kafka给自身的定位并不只是一个消息系统，而是通过发布订阅消息这种机制实现了流平台。

其实不管kafka给自己的定位如何，他都逃脱不了发布订阅消息的底层机制。本文讲解的重点，也是kafka发布订阅消息的特性。

Kafka和大多数消息系统一样，搭建好kafka集群后，生产者向特定的topic生产消息，而消费者通过订阅topic，能够准实时的拉取到该topic新消息，进行消费。如下图：
![](http://ww1.sinaimg.cn/large/006y8mN6ly1g68ysmkns1j31bg0ca74v.jpg)


Kafka特性

kafka和有以下主要的特性：

	1、消息持久化
	2、高吞吐量
	3、可扩展性

尤其是高吞吐量，是他的最大卖点。kafka之所以能够实现高吞吐量，是基于他自身优良的设计，及集群的可扩展性。后面章节会展开来分析。

Kafka应用场景

	1、消息系统
	2、日志系统
	3、流处理
小结：通过本章学习，可以掌握kafka的定位及其特性，了解消息系统的基本运作方式。以及kafka的应用场景。下面一章我们将通过安装和使用kafka，对kafka有近一步直观的认知。

下一步：开始[《kafka安装和使用》](https://lianxiaobao.github.io/2019/08/23/Apache-Kafka%E5%AE%89%E8%A3%85%E5%92%8C%E4%BD%BF%E7%94%A8/)的学习

本文转载于：https://blog.csdn.net/liyiming2017/article/details/82790040
 