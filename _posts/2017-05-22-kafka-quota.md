---
layout: post
title:  "使用kafka的配额管理实现客户端的限流"
categories: kafka
tags: kafka配额, kafka限流 
author: ShuxiaoW
---

* content
{:toc}

kafka支持配额管理，从而可以对Producer和Consumer的produce&fetch操作进行流量限制，防止个别业务压爆服务器。本文主要介绍如何使用kafka的配额管理功能。




## Kafka Quatas简介

Kafa

配额管理所能配置的对象（或者说粒度）有3种：

- user + clientid
- user
- clientid

这3种都是对接入的client的身份进行的认定方式。其中，clientid是每个接入kafka集群的client的一个身份标志，在ProduceRequest和FetchRequest中都需要带上；user只有在开启了身份认证的kafka集群才有。

可配置的选项包括：

- `producer_byte_rate`。发布者单位时间（每秒）内可以发布到**单台broker**的字节数。
- `consumer_byte_rate`。消费者单位时间（每秒）内可以从**单台broker**拉取的字节数。

可以使用kafka安装包中自带的脚本来配置配额：

```sh
# 1. 配置user+clientid。例如，user为"user1"，clientid为"clientA"。
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type users --entity-name user1 --entity-type clients --entity-name clientA

# 2. 配置user。例如，user为"user1"
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type users --entity-name user1

# 3. 配置client-id。例如，client-id为"clientA"
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type clients --entity-name clientA
```

**优先级**

如果对一个user既配置了user+clientid，又配置了user，那么配额是如何生效的呢？

## 实现原理

有关kafka quotas的一些实现细节，可以参考下面这篇文章：

[KIP-13 - Quotas](https://cwiki.apache.org/confluence/display/KAFKA/KIP-13+-+Quotas)

## 测试
