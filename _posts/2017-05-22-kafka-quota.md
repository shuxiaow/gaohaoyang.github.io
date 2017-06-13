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

## 如何配置

可以通过两种方式来作配额管理：

- 在配置文件中指定所有client-id的统一配额。
- 动态修改zookeeper中相关znode的值，可以配置指定`client-id`的配额。

使用第一种方式，必须重启broker，而且还不能针对特定`client-id`设置。所以，推荐大家使用第二种方式。

### 使用官方脚本修改配额

kafka官方的二进制包中，包含了一个脚本`bin/kafka-configs.sh`，支持针对`user`，`client-id`，`(user,client-id)`等三种纬度设置配额（也是通过修改zk来实现的）。

1. 配置user+clientid。例如，user为"user1"，clientid为"clientA"。
```sh
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type users --entity-name user1 --entity-type clients --entity-name clientA
```

2. 配置user。例如，user为"user1"
```sh
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type users --entity-name user1
```

3. 配置client-id。例如，client-id为"clientA"
```sh
bin/kafka-configs.sh  --zookeeper localhost:2181 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-type clients --entity-name clientA
```

### 直接写zk来修改配额

如果我们希望能够在代码里面直接写zk来实现配额管理的话，那要怎样操作呢？

假定我们在启动kafka时指定的zookeeper目录是`kafka_rootdir`。

1. 配置user+clientid。例如，针对"user1"，"clientA"的配额是10MB/sec，其它clientid的默认配额是5MB/sec。
    - znode: `${kafka_rootdir}/config/users/user1/clients/clientid`; value: `{"version":1,"config":{"producer_byte_rate":"10485760","consumer_byte_rate":"10485760"}}`
    - znode: `{kafka_rootdir}/config/users/user1/clients/<default>`; value: `{"version":1,"config":{"producer_byte_rate":"5242880","consumer_byte_rate":"5242880"}}`
2. 配置user。例如，"user2"的配额是1MB/sec，其它user的默认配额是5MB/sec。
    - znode: `${kafka_rootdir}/config/users/user1`; value: `{"version":1,"config":{"producer_byte_rate":"1048576","consumer_byte_rate":"1048576"}}`
    - znode: `${kafka_rootdir/config/users/<default>`; value: `{"version":1,"config":{"producer_byte_rate":"5242880","consumer_byte_rate":"5242880"}}`
3. 配置client-id。例如，"clientB"的配额是2MB/sec，其它clientid的默认配额是1MB/sec。
    - znode:`${kafka_rootdir}/config/clients/clientB'; value: `{"version":1,"config":{"producer_byte_rate":"2097152","consumer_byte_rate":"2097152"}}`
    - znode:`${kafka_rootdir}/config/clients/<default>`; value: `{"version":1,"config":{"producer_byte_rate":"1048576","consumer_byte_rate":"1048576"}}`

无论是使用官方的脚本工具，还是自己写zookeeper，最终都是将配置写入到zk的相应znode。所有的broker都会watch这些znode，在数据发生变更时，重新获取配额值并及时生效。为了降低配额管理的复杂度和准确度，kafka中每个broker各自管理配额。所以，上面我们配置的那些额度值都是单台broker所允许的额度值。

## 优先级

首先，我们需要明白，kafka在管理配额的时候，是以“组”的概念来管理的。而管理的对象，则是producer或consumer到broker的一条条的TCP连接。

那么在进行额度管理的时候，kafka首先需要确认，这条连接属于哪个“组”，进而确定当前连接是否超过了所属“组”的总额度。

在进行“组”判定的时候，依照以下的优先级顺序依次判定：

```sh
/config/users/<user>/clients/<client-id>
/config/users/<user>/clients/<default>
/config/users/<user>
/config/users/<default>/clients/<client-id>
/config/users/<default>/clients/<default>
/config/users/<default>
/config/clients/<client-id>
/config/clients/<default>
```

一旦找到了符合的“组”，即中止判定过程。

## 超额处理

如果连接超过了配额值会怎么样呢？kafka给出的处理方式是：延时回复给业务方，不使用特定返回码。

具体到producer还是consumer，处理方式又有所不同：

- Producer。如果Producer超额了，先把数据append到log文件，再计算延时时间，并在ProduceResponse的ThrottleTime字段填上延时的时间（v2，只在0.10.0版本以上支持）。
- Consumer。如果Consumer超额了，先计算延时时间，在延时到期后再去从log读取数据并返回给Consumer。否则无法起到限制对文件系统的读蜂拥。在v1（0.9.0以上版本）和v2版本的FetchResponse中有ThrottleTime字段，表示因为超过配额而延时了多久。
