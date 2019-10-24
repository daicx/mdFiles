---
title: 消息队列之Kafka
date: 2019-05-15 00:17:49
tags:
- kafka
categories:
- java 中间件
- kafka
---

# 消息队列之Kafka

## 1.简介

Kafka是一个基于分布式的消息发布-订阅系统.

## 2.专用术语

### ① Broker(代理)

kafka包含一个或者多个服务器,一个服务器服务器被称为一个Broker.

### ②Topic(主题)

发布到Kafka集群的消息都有一个类别,这个类别成为Topic.

### ③Partition(分区)

物理上的概念,每个Topic包含一个或者多个Partition.

### ④Replica(副本)

每个partition有多个副本,存储在不同的broker上,保证高可用.

### ⑤Segment(片段)

partition是由多个segment组成,每个segment存储着message信息

### ⑥Message(消息)

基本的通信单位,由一个key,一个value和时间戳组成.

### ⑦Producer(生产者)

消息生产者,向broker发送消息的客户端

### ⑧Comsumer(消费者)

消息消费者,从broker读取消息的客户端.

### ⑨ComsumerGroup(消费者群组)

每个comsumer属于一个特定的Comsumer Group,一个消息可以发送到多个Comsumer Group,但是一个Comsumer Group中只能有一个Comsumer消费改消息.

## 3.分布式订阅模型

生产者->brocker->消费者

## 4.消费模型

采用拉取模型,消费的进度和速度,由消费者决定.

## 5.高可用模型(多副本)

1.对于每一个Topic,我们都可以设置包含有多少个Partition,每个Partition包含Topic的一部分内容.

2.每个broker都会存储一些Partition,因此在kafka集群中,实现了Topic数据的分布式存储.

<iframe frameborder="0" style="width:100%;height:593px;" src="https://www.draw.io/?lightbox=1&highlight=0000ff&layers=1&nav=1&title=Kafka%E7%9A%84Topic%E5%AD%98%E5%82%A8.drawio#R7ZhLU9swEIB%2FjWbaAx2%2FLR3jJLRMS8s09MFRiRVb4FiuopCkv76SLcc2djIUMDATcgBp9d5vtbsysIeLzUeOs%2FichSQBlhFugD0CluW7tvyrBNtC4Bp%2BIYg4DQuRWQkm9C%2FRQkNLVzQky0ZHwVgiaNYUzliakployDDnbN3sNmdJc9UMR6QlmMxw0pb%2BoqGICym0%2FEr%2BidAoLlc2PVS0LHDZWZ9kGeOQrWsiewzsIWdMFKXFZkgSpbtSL8W40z2tu41xkor7DBjwk7NvP%2BB3eHZufd1k5Ay61yemxnOLk5U%2B8Q2e32AwRgAFAHpg7INgDAaOPoTYlprhbJWGRE1uAjtYx1SQSYZnqnUtTUHKYrFIdPOcpeIUL2iizOAn4SFOsRZr5qZUSBDiZbyb8pZwQSWJQUKjVMqmTAi2UKNokgxZwni%2BEXs%2Bn80QkvKl4OyG1Fpsz0a2nC7Qh5QTks1e9Zk7KNKYCVsQwbeySznA0xy1IVtQ19eVWexYxzWTsEtTxtoUo93cFS1Z0MD%2BAx5qsVO0IAIDpArIyPm5AMKyMATB4HEgH0YIzX01qF9CqAnIdNuAHNgByOwNkHdA14Y8NOMiZhFLcfKFsUwr9ZoIsdWXAq8Ea%2BqfbKj4rYZ%2FcHXtqtYy2uiZ88q2rKTyMLVBqnpVb6uG5bVyXLF7ErY85B0i8oRsxWfkkJ%2FRrhvziIgD%2FaxuwpwkWNDb5j6eHJffwqXOPtHVlKXyX%2FBG8DBB5yUJwjeCjyfoviRBsxXTLllGZ8%2BSfah6I69Qv3Y0MwyIDaMrmhmG6xsVtUdFM8tvhrOOdMM0OqKZ21cws1pkLrAM%2F4Ky1HxNeDwDI9PvwmONfO%2Bp8Jj2nWzDvycfpy8%2B7Ux%2Bqs7PgeUlcu1gqkqRKr1TueBgBBDMk0ILDOAFZxrl%2B3b%2F56HbpEjyX98p4%2B6ZqSnaRgfFrqTe64uis%2F%2BWWcd3y3Ze79XcMnc%2FH%2Fv4%2BDivzguWF7r%2BKNYOTWmuwcf7s2Jlw8ky17F8HRsWzDZVY%2BkEPxefRRwQIACDIilR3nMM0GnjtY0C9cFErynPUCy7x5NK1YumGXAid4KneQdFKWM0FbmW3AC4IylRCelSW4Sq6sd3Quai400uVFIbLKXZ0TS6zDNc6WR68Z4mbMHfOco6fKs3%2BO3s8SEx0DruGIheOgaa7VTzIRjto8bodD0YngijrFbfrfO22sd%2Fe%2FwP"></iframe>

为了保证,在其中一台broker宕机后,数据不会丢失,采用了常见的分布式处理方式:多副本机制.在Kafka集群中,每个Partition都会有多个副本,包括一个主副本Leader和0或者多个子副本Follower.这些副本分布在不同的broker上,从而保证了数据的完整性.

3.多副本之间的数据同步

副本之间是如何保证同步的呢?

在生产者和消费者操作Kafka的时候,只有主副本Leader提供读写服务.其他Follower副本会不停的对Leader副本发送请求,拉取最新的数据,然后写到磁盘.

<iframe frameborder="0" style="width:100%;height:433px;" src="https://www.draw.io/?lightbox=1&highlight=0000ff&layers=1&nav=1&title=Kafka%E5%89%AF%E6%9C%AC%E6%9C%BA%E5%88%B6.drawio#R7VhLc9owEP41mmkP6dgWfuhog2k6TXtoOmlyVGzZViIsKkSA%2FvpKtoxxDEzSCZBJemG0364eu99qtRjA4WT5WeBp8Y2nhAHHSpcAjoDj%2BC5UvxpY1YBr%2BTWQC5rWkN0Cl%2FQPMaBl0DlNyaxjKDlnkk67YMLLkiSyg2Eh%2BKJrlnHW3XWKc9IDLhPM%2BugvmsqiRgPHb%2FFzQvOi2dn2UK2Z4MbYeDIrcMoXGxCMARwKzmU9miyHhOnYNXGp5413aNcHE6SUT5kw%2F%2F7jevQlje3zq5s78T1eXGB0Zht6HjCbG4%2B%2F4uwegxgBFIHAA7EPohiEA%2BOEXDWREXxepkQvbgMYLQoqyeUUJ1q7UKmgsEJOmFFnvJRjPKFMp8EVESkusYEN57YKSPRAhKQq%2BCGjeanAWy4ln2hDytiQMy6qvWGGMl9vHc2k4PdkQwM9iKDWGL%2FUgmS5M2L2mgeVv4RPiBQrZWImQMtEZ7Umt5YXbSY4gaG32MgCaJtwYZN9%2BXrtliA1MBw9gy%2FUo4GkKl%2BNyIUseM5LzOIWjVqiLCW1NhecTw09d0TKlSECzyXvkkeWVF5vjG%2F0Up9cI42WZuVKWDVCqdxdT9JCPct3G7mdV0nNxNpB7dV%2B1lQQ%2BFwkZF9ym3qBRU7kHjt3exYIwrCkD91zvDijTaF7DZQ6%2F8bpK6R0cFJKezVVuSqppLwEjsfUuaNboUa5Hn24IDgl4mNfc5SK%2B7iwZkmC0MELq2d1Cuv6jdworLazpbA2816cMqdHWcOHDliHBu%2F3nDeKs1kVylAZQG%2B6bJUNh%2BYxHYAIgSACsQsCBMIxiD2AhiAcmkEUVqoARF6zr3Kj3npHMqjoyy7jgqjT4NvKQF%2B7KaelrALlRsAdKUTXgZkhX4vmlWUkk1seX6lrSTRTGUbL%2FGdVWNS9epEEUG1gJwFcp%2F%2Bwoi38O4fif%2FCsKzvmjPHFO7%2B09uDUl9b9T9pzSYMnr7T%2B3nan5OXpW9a1cPT2xntie7OD9OO0N17%2FrVR%2FEtEAoHH10oUg9EEcAPXPKHBPcbWShLhZduirZcPX1sQEPWIWQoX3jbUO%2FuOwo17YB8dsHex%2Bu6%2B7OtXtqVZPd3VjgLyqz3NBZFWqEQjjt8WKa7vd5gD1P5V4R2VlS0f%2FDlnxuq%2B%2F5R6KFSW2nzIr3cb3YBj%2FBQ%3D%3D"></iframe>

4.ISR是什么?

ISR全称是“In-Sync Replicas",意思是保持同步的副本.意思上图中的Follower+Leader.

## 6.生产者的ACKS参数

acks是Producer设置的,选项有3个:0,1,all.含义是生产者的消息确认机制.

6.1 acks=0

当生产者把消息发出去之后,不管是否落到了那个broker的protition中,总是认为是成功了.

缺点:消息发出去,leader还没收到就宕机了,这条消息就丢失了.

6.2 acks=1(默认)

生产者把消息发出去,并且Leader收到并且保存到了磁盘,就认为是成功了.

缺点:Leader在保存后,发生了宕机,数据也会丢失.

6.3 acks=all

生产者发送消息后,Leader和Follower副本(即ISR列表)都将数据保存都了磁盘,就认为是成功了.

注意:需要ISR列表里至少有一个Follower副本.

## 7消费者

 **Consumers** Kafka提供了两套consumer api，分为high-level api和sample-api。Sample-api 是一个底层的API，它维持了一个和单一broker的连接，并且这个API是完全无状态的，每次请求都需要指定offset值，因此，这套API也是最灵活的。 
**在kafka中，当前读到哪条消息的offset值是由consumer来维护的**，因此，consumer可以自己决定如何读取kafka中的数据。**比如，consumer可以通过重设offset值来重新消费已消费过的数据。不管有没有被消费，kafka会保存数据一段时间，这个时间周期是可配置的，只有到了过期时间，kafka才会删除这些数据。（这一点与AMQ不一样，AMQ的message一般来说都是持久化到mysql中的，消费完的message会被delete掉）**
High-level API封装了对集群中一系列broker的访问，可以透明的消费一个topic。它自己维持了已消费消息的状态，即每次消费的都是下一个消息。 
High-level API还支持以组的形式消费topic，如果consumers有同一个组名，那么kafka就相当于一个队列消息服务，而各个consumer均衡的消费相应partition中的数据。若consumers有不同的组名，那么此时kafka就相当与一个广播服务，会把topic中的所有消息广播到每个consumer。 
High level api和Low level api是针对consumer而言的，和producer无关。 
High level api是consumer读的partition的offsite是存在zookeeper上。High level api 会启动另外一个线程去每隔一段时间，offsite自动同步到zookeeper上。换句话说，如果使用了 High level api， 每个message只能被读一次，一旦读了这条message之后，无论我consumer的处理是否ok。High level api的另外一个线程会自动的把offiste+1同步到zookeeper上。如果consumer读取数据出了问题，offsite也会在zookeeper上同步。因此，如果consumer处理失败了，会继续执行下一条。这往往是不对的行为。因此，Best Practice是一旦consumer处理失败，直接让整个conusmer group抛Exception终止，但是最后读的这一条数据是丢失了，因为在zookeeper里面的offsite已经+1了。等再次启动conusmer group的时候，已经从下一条开始读取处理了。 
Low level api是consumer读的partition的offsite在consumer自己的程序中维护。不会同步到zookeeper上。但是为了kafka manager能够方便的监控，一般也会手动的同步到zookeeper上。这样的好处是一旦读取某个message的consumer失败了，这条message的offsite我们自己维护，我们不会+1。下次再启动的时候，还会从这个offsite开始读。这样可以做到exactly once对于数据的准确性有保证。