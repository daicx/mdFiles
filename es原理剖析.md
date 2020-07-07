# ES原理剖析

es重度使用了文件系统缓存.

## es数据名词

1. Cluster:ES集群,由多个Node组成.
2. Node:节点,一个es实例就是一个node.
3. index:索引,一系类Documents的集合.对于存入的数据,都会有一个索引对应.
4. shard:分片.每个索引都会被分为多个分片,默认是5个.存储在不同的节点上.
5. replica:副本,容灾.默认会有5个副本.副本和分片不会出现在同一个node上.

## 新增数据处理流程

流程图

![image-20200707232912247](es%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/image-20200707232912247.png)

### 新增数据进内存Buffer

​		将新增的数据,进行分词切分,进行倒排索引,存入内存buffer,同时生成一份translog文件.

对于translog文件,每隔5s或者对于增删更新数据的请求,都会触发translog文件持久化到磁盘.

​		注:内存和cache,buffer的区别.

内存:程序运行的地点,与CPU沟通的桥梁.

cache:介于CPU与内存之间,CPU操作速度远大于内存,因此将内存中的数据提前放进cache,以便CPU处理上个指令后,可以立马执行下一个指令,提高CPU处理效率.

buffer:为了提高内存和磁盘的数据IO进行设计的.使IO可以批量操作,减少IO次数.

### 数据进入文件系统缓存refresh

​		默认每隔1s,会进行一次refresh操作,将内存buffer中的数据,生成一个segment文件,存到文件系统缓存.此时,里面数据就可以被搜索了.

es提供了主动刷新数据的接口:**/_refresh**.

如果感觉1s时间刷新太长,可以在更新完数据后,调用此接口进行数据刷新,使其可被搜索.如果在同步大量数据时,在这期间不关心是否可被搜索,提高同步效率,就使用 

```cpp
# curl -XPOST http://127.0.0.1:9200/索引_2020-04-01/_settings -d'
{ "refresh_interval": "10s" }
'
```

根据情况降低刷新次数.

或者可以在同步历史数据时,将refresh关闭

```cpp
# curl -XPUT http://127.0.0.1:9200/索引_2020-04-01 -d'
{
  "settings" : {
    "refresh_interval": "-1"
  }
}'
```

等同步完成后,在手动执行refresh,使其可被搜索.

```cpp
# curl -XPOST http://127.0.0.1:9200/索引_2020-04-01/_refresh
```

### 数据进入磁盘flush

默认每隔30分钟,或者translog文件达到了512MB,就执行一次flush操作.把文件系统缓存中的segment文件,持久化到磁盘.完成后,同时清空translog文件.

对于时间间隔和大小,可以进行设置.

```
//flush时间间隔
index.translog.flush_threshold_period
//触发flush文件大小
index.translog.flush_threshold_size
```

这样,数据就完成了存入.

## 数据的维护

数据存到系统中了,就需要对数据进行维护,来解决在录入数据时产生的负作用,比如,refresh会每秒生成一个segment文件,如果要对数据检索,就要挨个检查这些文件,显然是及其浪费资源的.为了解决这个问题,使用文件归并策略.

### Segment归并

在文件系统缓存中,会根据一些策略对其中的segment文件开启一些线程进行归并.归并完成后刷新到磁盘,将请求从小segment切换到归并后的segment后,删除旧的segment.

1. index.merge.policy.floor_segment

   默认2MB,小于这个大小的segment会优先被归并.

2. index.merge.policy.max_merge_at_once

   默认10个.一次默认归并多少个segment.

3. index.merge.policy.max_merge_at_once_explicit

   默认30个.在forcemerge时,一次会归并多少个segment.

4. index.merge.policy.max_merged_segment

   默认5GB.大于这个的segment,不会参与归并.forcemerge除外.

由于归并非常消耗IO和CPU,因此,默认会使用 Math.min(3,Runtime.getRuntime().availableProcessors() / 2)个线程来进行归并.CPU核数大于6时,会启动3个归并线程.

```
//最大归并线程数量
index.merge.scheduler.max_thread_count
```

以上是数据分配到了节点上后,节点进行的操作.那么在分布式es中,是如何判断数据该存到哪个节点上呢?

## 新增数据到节点

对于每条数据,都会给其分配**_id**,而数据分配到哪个节点就是根据这个参数计算.

```
//节点0,1,2,3        主分片数
shard = hash(_id) % number_of_primary_shards
```

默认情况下,每个索引会建立5个主分片.

![image-20200708010232348](es%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/image-20200708010232348.png)

在客户端请求写入数据时,流程如上图.

1. 请求由master节点处理.
2. mater是node1.
3. master经过_id计算shard,发现算出来是shard1,则根据集群中的信息寻找shard1存在了node2节点上,将请求转发到node2.
4. node2数据处理完后,告诉node3进行副本数据同步.
5. node3数据备份完成后,告知node2.
6. node2返回master节点node1,从而通知客户端数据写入成功

对于副本,在同步大文件时,可以先取消副本,然后在同步完成后开启,从而减少segment归并时产生的消耗.

```cpp
# curl -XPUT http://127.0.0.1:9200/索引_2020-04-01/_settings -d '{
    "index": { "number_of_replicas" : 0 }
}'
```

## shard分配策略

shard分配到哪个节点上,通常由es自动决定.以下会触发分配动作.

1. 新索引的生成.
2. 索引的删除.
3. 新增副本分片.
4. 节点增减引发的数据均衡.

## 数据均衡

为了保护安全,每隔30s会检查下各节点磁盘使用量.如果超过了85%,新索引分配就不会在分配到这个节点.超过了90%,就会触发该节点上现存分片的数据均衡,把数据挪到其他节点上去.

```bash
# curl -XPUT localhost:9200/_cluster/settings -d '{
    "transient" : {
        "cluster.routing.allocation.disk.watermark.low" : "85%",
        "cluster.routing.allocation.disk.watermark.high" : "10gb",
        "cluster.info.update.interval" : "1m"
    }
}'
```

## 分片移动

因为负载过高，磁盘利用率过高，服务器下线，更换磁盘等原因，可以会需要从节点上移走部分分片：

```rust
curl -XPOST 127.0.0.1:9200/_cluster/reroute -d '{
  "commands" : [ {
        "move" :
            {
              "index" : "索引_2020-04-01", "shard" : 0, "from_node" : "10.19.0.81", "to_node" : "10.19.0.104"
            }
        }
  ]
}
```

## 节点下线

Elasticsearch 集群就会自动把这个 IP 上的所有分片，都自动转移到其他节点上。等到转移完成，这个空节点就可以毫无影响的下线了。

```rust
curl -XPUT 127.0.0.1:9200/_cluster/settings -d '{
  "transient" :{
      "cluster.routing.allocation.exclude._ip" : "10.0.0.1"
   }
}'
```

## 冷热数据的读写分离

Elasticsearch 集群一个比较突出的问题是: 用户做一次大的查询的时候, 非常大量的读 IO 以及聚合计算导致机器 Load 升高, CPU 使用率上升, 会影响阻塞到新数据的写入, 这个过程甚至会持续几分钟。所以，可能需要仿照 MySQL 集群一样，做读写分离。

### 实施方案

1. N 台机器做热数据的存储, 上面只放当天的数据。这 N 台热数据节点上面的 elasticsearc.yml 中配置 `node.attr.tag: hot`

2. 之前的数据放在另外的 M 台机器上。这 M 台冷数据节点中配置 `node.attr.tag: stale`

3. 模板中控制对新建索引添加 hot 标签：

```json
{
    "order" : 0,
    "template" : "*",
    "settings" : {
      "index.routing.allocation.include.tag" : "hot"
    }
}
```

4. 每天计划任务更新索引的配置, 将 tag 更改为 stale, 索引会自动迁移到 M 台冷数据节点

```cpp
curl -XPUT http://127.0.0.1:9200/indexname/_settings -d'
{
   "index": {
      "routing": {
         "allocation": {
            "include": {
               "tag": "stale"
            }
         }
     }
   }
}'
```

这样，写操作集中在 N 台热数据节点上，大范围的读操作集中在 M 台冷数据节点上。避免了堵塞影响。

## 集群自动发现

### unicast 方式

ES 从 2.0 版本开始，默认的自动发现方式改为了单播(unicast)方式。配置里提供几台节点的地址，ES 将其视作 gossip router 角色，借以完成集群的发现。由于这只是 ES 内一个很小的功能，所以 gossip router 角色并不需要单独配置，每个 ES 节点都可以担任。所以，采用单播方式的集群，各节点都配置相同的几个节点列表作为 router 即可。

此外，考虑到节点有时候因为高负载，慢 GC 等原因可能会有偶尔没及时响应 ping 包的可能，一般建议稍微加大 Fault Detection 的超时时间。

同样基于安全考虑做的变更还有监听的主机名。现在默认只监听本地 lo 网卡上。所以正式环境上需要修改配置为监听具体的网卡。

```css
network.host: "192.168.0.2" 
discovery.zen.minimum_master_nodes: 3
discovery.zen.ping_timeout: 100s
discovery.zen.fd.ping_timeout: 100s
discovery.zen.ping.unicast.hosts: ["10.19.0.97","10.19.0.98","10.19.0.99","10.19.0.100"]
```

上面的配置中，两个 timeout 可能会让人有所迷惑。这里的 **fd** 是 fault detection 的缩写。也就是说：

- discovery.zen.ping_timeout 参数仅在加入或者选举 master 主节点的时候才起作用；
- discovery.zen.fd.ping_timeout 参数则在稳定运行的集群中，master 检测所有节点，以及节点检测 master 是否畅通时长期有用。

既然是长期有用，自然还有运行间隔和重试的配置，也可以根据实际情况调整：

```css
discovery.zen.fd.ping_interval: 10s
discovery.zen.fd.ping_retries: 10
```

## es优化

### Filesystem Cache

es重度使用了Filesystem Cache,如果分配更多的内存,使大多数segment都在Filesystem Cache中,则查询效率最大化.

案例:某个公司 ES 节点有 3 台机器，每台机器看起来内存很多 64G，总内存就是 64 * 3 = 192G。每台机器给 ES JVM Heap 是 32G，那么剩下来留给 Filesystem Cache 的就是每台机器才 32G，总共集群里给 Filesystem Cache 的就是 32 * 3 = 96G 内存。而此时，整个磁盘上索引数据文件，在 3 台机器上一共占用了 1T 的磁盘容量，ES 数据量是 1T，那么每台机器的数据量是 300G。这样性能好吗?

　　Filesystem Cache 的内存才 100G，十分之一的数据可以放内存，其他的都在磁盘，然后你执行搜索操作，大部分操作都是走磁盘，性能肯定差。

　　归根结底，你要让 ES 性能好，最佳的情况下，就是你的机器的内存，至少可以容纳你的总数据量的一半。

　　根据我们自己的生产环境实践经验，最佳的情况下，是仅仅在 ES 中就存少量的数据。

　　也就是说，你要用来搜索的那些索引，如果内存留给 Filesystem Cache 的是 100G，那么你就将索引数据控制在 100G 以内。这样的话，你的数据几乎全部走内存来搜索，性能非常之高，一般可以在1秒以内。

　　比如说你现在有一行数据：id，name，age .... 30 个字段。但是你现在搜索，只需要根据 id，name，age 三个字段来搜索。

　　如果你傻乎乎往 ES 里写入一行数据所有的字段，就会导致 90% 的数据是不用来搜索的。

　　但是呢，这些数据硬是占据了 ES 机器上的 Filesystem Cache 的空间，单条数据的数据量越大，就会导致 Filesystem Cahce 能缓存的数据就越少。

　　其实，仅仅写入 ES 中要用来检索的少数几个字段就可以了，比如说就写入 es id，name，age 三个字段。

　　然后你可以把其他的字段数据存在 MySQL/HBase 里，我们一般是建议用 ES + HBase 这么一个架构。

　　HBase是列式数据库，其特点是适用于海量数据的在线存储，就是对 HBase 可以写入海量数据，但是不要做复杂的搜索，做很简单的一些根据 id 或者范围进行查询的这么一个操作就可以了。

　　从 ES 中根据 name 和 age 去搜索，拿到的结果可能就 20 个 doc id，然后根据 doc id 到 HBase 里去查询每个 doc id 对应的完整的数据，给查出来，再返回给前端。

　　而写入 ES 的数据最好小于等于，或者是略微大于 ES 的 Filesystem Cache 的内存容量。

　　然后你从 ES 检索可能就花费 20ms，然后再根据 ES 返回的 id 去 HBase 里查询，查 20 条数据，可能也就耗费个 30ms。

　　如果你像原来那么玩儿，1T 数据都放 ES，可能会每次查询都是 5~10s，而现在性能就会很高，每次查询就是 50ms。

### 数据预热

　　假如你就按照上述的方案去做了，ES 集群中每个机器写入的数据量还是超过了 Filesystem Cache 一倍。

　　比如说你写入一台机器 60G 数据，结果 Filesystem Cache 就 30G，还是有 30G 数据留在了磁盘上。

　　这种情况下，其实可以做数据预热。举个例子，拿微博来说，你可以把一些大 V，平时看的人很多的数据，提前在后台搞个系统。

　　然后每隔一会儿，自己的后台系统去搜索一下热数据，刷到 Filesystem Cache 里去，后面用户实际上来看这个热数据的时候，他们就是直接从内存里搜索了，很快。

　　或者是电商，你可以将平时查看最多的一些商品，比如说 iPhone 8，热数据提前后台搞个程序，每隔 1 分钟自己主动访问一次，刷到 Filesystem Cache 里去。

　　总之，就是对于那些你觉得比较热的、经常会有人访问的数据，最好做一个专门的缓存预热子系统。

　　然后对热数据每隔一段时间，就提前访问一下，让数据进入 Filesystem Cache 里面去。这样下次别人访问的时候，性能一定会好很多。

### 冷热分离

　　ES 可以做类似于 MySQL 的水平拆分，就是说将大量的访问很少、频率很低的数据，单独写一个索引，然后将访问很频繁的热数据单独写一个索引。

　　最好是将冷数据写入一个索引中，然后热数据写入另外一个索引中，这样可以确保热数据在被预热之后，尽量都让他们留在 Filesystem OS Cache 里，别让冷数据给冲刷掉。

　　还是来一个例子，假设你有 6 台机器，2 个索引，一个放冷数据，一个放热数据，每个索引 3 个 Shard。3 台机器放热数据 Index，另外 3 台机器放冷数据 Index。

　　这样的话，你大量的时间是在访问热数据 Index，热数据可能就占总数据量的 10%，此时数据量很少，几乎全都保留在 Filesystem Cache 里面了，就可以确保热数据的访问性能是很高的。

　　但是对于冷数据而言，是在别的 Index 里的，跟热数据 Index 不在相同的机器上，大家互相之间都没什么联系了。

　　如果有人访问冷数据，可能大量数据是在磁盘上的，此时性能差点，就 10% 的人去访问冷数据，90% 的人在访问热数据，也无所谓了。

### Scroll API分页