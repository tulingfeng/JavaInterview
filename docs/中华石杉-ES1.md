## 02

**使用数据库搜索**

```text
1.搜索字段的长度可能会很长，每次都要对文本的所有信息进行判断
2.不能将搜索词分开，比如说，搜索生化机，找不到生化危机。
```

**lucene**

就是一个jar包，里面包含了封装好的各种建立倒排索引，以及进行搜索的代码，包括各种算法。

**什么是elasticsearch**

从解决痛点出发：

```text
1.自动维护数据的分布到多个节点的索引的建立，还有搜索请求分不到多个节点执行。
2.自动维护数据的冗余副本，保证说一些机器宕机，数据不丢失。
3.封装了更高级的功能，可以更快速的开发应用。
```



## 03 

**elasticsearch的功能**

1.分布式的搜索引擎和数据分析引擎

2.全文检索、结构化检索、数据分析

全文搜索：`select * from products where product_name like "%牙膏%"`

结构化检索：`select * from products where category_id="日化用品"`

数据分析：`select category_id,count(*) from products group by category_id `

3.对海量数据进行近实时的处理

分布式、近实时（秒级别）



## 04

elasticsearch基于lucene，隐藏复杂性，使用简单易用的rest api。

>分布式的文档存储引擎
>
>分布式的搜索引擎和分析引擎
>
>分布式，支持PB数据

**elasticsearch核心概念**

```text
1.NRT(近实时)，延迟大概1s，refresh时间
2.Cluster:集群
3.Node:节点
4.Document:文档，es中的最小数据单元，一条客户数据，通常用json表示
5.Index:索引，可以包含很多Document
6.Shard:一个index分成很多个shard
好处：横向扩展、数据分布在多个shard，都会在多台服务器上并行分布式执行，提升吞吐量和性能
Primary Shard
7.Replica:Replica Shard，高可用、提升搜索请求(可以把请求发送到replica)的吞吐量和性能。
```



## 06

elasticsearch 7.0以前搭建配置：

```yaml
bootstrap.memory_lock=true  # 避免jvm进行堆内堆外交换，影响es性能
discovery.zen.ping.unicast.hosts=elasticsearch 
# 单播机制，配置静态主机列表用作种子节点，使用基于tcp的transport模块，天然异步
```

elasticsearch 7.0以后搭建配置：

```yaml
discovery.seed_hosts:es02,es03   # 提供集群内部可能活着的其他节点，也可以提供seed_providers:file去动态管理
cluster.initial_master_nodes:es01,es02,es03 # 提供能成为主节点候选的节点
bootstrap.memory_lock:true  # 同上
```

常用api：

```text
1.查看集群状态
GET /_cat/health?pretty
2.查看索引情况
GET /_cat/indices?pretty
3.插入数据
PUT /test/_doc/1
{
    "name":"test"
}
注意：6.x系列取消了PUT /index/type/id这种形式。
去掉type目的是不让在不同index下的相同field发生冲突。
4.更新数据
POST /test/_doc/1/_update
{
  "doc":{   # 这里的doc类型必须加上
    "name":"xiaoming"
  }
}
```



```text
docker-compose安装报错：
# [[/usr/share/elasticsearch/data/docker-cluster]] with lock id [0]; maybe these locations are not writable or multiple nodes were started without increasing [node.max_local_storage_nodes] (was [1])?

需要设置参数：node.max_local_storage_nodes=2(如果集群是2节点的话)

安装细节：
1.必须将es、kibana放在一个局域网下
2.kibana可以依赖es
environment:
  SERVER_NAME: localhost
  ELASTICSEARCH_URL: http://elasticsearch:9200/
3.改变挂载文件夹权限及用户组
```



## 07

搜索方式：

1.query string search

```text
GET /test/_search 

{
  "took": 15,  # 耗时
  "timed_out": false, # 是否超时
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,  # 查询结果数量
    "max_score": 1, # document对一个search的相关度的匹配分数，越相关，就越匹配，分数就越高。
    "hits": [  # 命中的document的详细信息
      {
        "_index": "test",
        "_type": "_doc",
        "_id": "1",
        "_score": 1,  
        "_source": {
          "name": "xiaoming"
        }
      }
    ]
  }
}

# 针对某字段进行搜索排序
GET /test_price/_search?sort=age:desc
```



2.query DSL

```text
GET /test_price/_search
{
  "query":{
    "match_all": {}
  }
  "from": 1,
  "size": 20  # 分页大小
}

# 按某字段降序排序
GET /test_price/_search
{
  "query":{
    "match_all": {}
  },
  "sort":[{"age":"desc"}]
}
```



3.query filter

```text
GET /test_price/_search
{
  "query":{
    "bool":{
      "must":{
        "match":{"name":"tutu"}  # match name text，但是name也必须分词才能进行match
      },
      "filter": {
          "range":{
              "age":{"gt": 10}
          }
      }
    }
  }
}
```



4.phrase search

```text
GET /test_price/_search
{
  "query":{
    "match_phrase": {
      "name": "xiaohong"  # 必须包含这个短语
    }
  }
}
```



## 08 聚合

1.聚合

```text
GET /test_price/_search
{
  "size": 0,   # 去掉hits信息
  "aggs": {
    "aggs_groupy": {
      "terms": {
        "field": "name.keyword",
        "size": 10
      }
    }
  }
}
在聚合字段之前，需要修改相关字段：
PUT /test_price/_doc/_mapping
{
  "_doc":{
    "properties":{
    "name":{
      "type":"text",
      "fielddata":true
      }
    }
  }
}

# result
"aggregations": {
    "aggs_groupy": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "tutu ars",
          "doc_count": 2
        },
        {
          "key": "tutu",
          "doc_count": 1
        },
        {
          "key": "xiaohong",
          "doc_count": 1
        },
        {
          "key": "xiaoli",
          "doc_count": 1
        },
        {
          "key": "xiaoming",
          "doc_count": 1
        }
      ]
    }
  }
```

2.query+聚合

```text
GET /test_price/_search
{
  "size":0,
  "query":{   # query 与aggs并列
    "match": {
      "name": "tutu"
    }
  },
  "aggs": {
    "aggs_groupy": {
      "terms": {
        "field": "name.keyword",
        "size": 10
      }
    }
  }
}
```

3.分组+聚合

```text
GET /test_price/_search
{
  "size":0,
  "aggs": {
    "aggs_groupy": {
      "terms": {
        "field": "name",
        "size": 10
      },
      "aggs": {
        "avg_ages": {
          "avg": {
            "field": "age"
          }
        }
      }
    }
  }
}
```

4.分组+聚合+排序

```text
GET /test_price/_search
{
  "size":0,
  "aggs": {
    "aggs_groupy": {
      "terms": {
        "field": "name",
        "order": {
          "avg_ages": "desc"  # 按照聚合组名进行排序
        }
      },
      "aggs": {
        "avg_ages": {
          "avg": {
            "field": "age"
          }
        }
      }
    }
  }
```

5.分桶+聚合+算平均

```text
GET /test_price/_search
{
  "size":0,
  "aggs": {
    "gropu_by_ages": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 0,
            "to": 10
          },
          {
            "from":10,
            "to":20
          },
          {
            "from":20,
            "to": 50
          }
        ]
      },
      "aggs": {
    "aggs_groupy_ages": {
      "terms": {
        "field": "name"
      },
      "aggs": {
        "avg_ages": {
          "avg": {
            "field": "age"
              }
            }
          }
        }
      }
    }
  }
}

# result
"aggregations": {
    "gropu_by_ages": {
      "buckets": [
        {
          "key": "0.0-10.0",
          "from": 0,
          "to": 10,
          "doc_count": 0,
          "aggs_groupy_ages": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": []
          }
        },
        {
          "key": "10.0-20.0",
          "from": 10,
          "to": 20,
          "doc_count": 4,
          "aggs_groupy_ages": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "ars",
                "doc_count": 2,
                "avg_ages": {
                  "value": 16
                }
              },
              {
                "key": "tutu",
                "doc_count": 2,
                "avg_ages": {
                  "value": 16
                }
              },
              {
                "key": "xiaohong",
                "doc_count": 1,
                "avg_ages": {
                  "value": 18
                }
              },
              {
                "key": "xiaoli",
                "doc_count": 1,
                "avg_ages": {
                  "value": 10
                }
              }
            ]
          }
        },
        {
          "key": "20.0-50.0",
          "from": 20,
          "to": 50,
          "doc_count": 2,
          "aggs_groupy_ages": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "tutu",
                "doc_count": 1,
                "avg_ages": {
                  "value": 25
                }
              },
              {
                "key": "xiaoming",
                "doc_count": 1,
                "avg_ages": {
                  "value": 20
                }
              }
            ]
          }
        }
      ]
    }
  }
```



## 09 分布式架构

分布式透明隐藏性

扩容方案：

```text
1.垂直扩容：直接买机器换掉老节点
2.水平扩容：在集群内加新节点  # 一般选取这个方案
```

增加减少节点会出现rebalance。

某些节点负载总是重一些，增加节点会自动进行rebalance。

**master节点**

```text
1.管理es集群的元数据：索引的创建和删除，维护索引数据；节点增加和减少；维护集群元数据
2.master不会承载所有的请求，所以不存在单点瓶颈。
如果master节点不存储数据，配置相应可以降低一些。
```

**节点对等的分布式架构**

请求发给谁都可以。



## 10 Shard&Replica

```text
1.shard 是lucene实例，具有完整的创建索引和处理请求的能力。
2.replica是shard的副本，容错和承担读请求。
3.replica可以随时修改。
4.shard和replica不能放到同一个node上。
```



## 12 横向扩容，如何超出扩容极限

shard/replica自动转移到新加节点，意味着每个shard可以占用节点上的更多资源。IO/CPU/Memory，整个系统会更好。

如果：只有6shard 9node怎么分配，让系统性能更好？

```text
增加replica数量
```

扩容跟容错性是需要一起考虑的。



## 13 容错机制

```text
1.master选举
master宕机 重新选举。
2.replica
丢失的primary shard 的replica 提升为primary shard.
3.重启故障的node
```



## 14 document核心元数据

```text
1._type:5.4之后只有_doc这种类型
2._id:代表document的唯一表示
3._index:代表document放在哪个索引里面
类似的数据放在一个index，因为这些数据的功能大概率是一样的。相同的数据放在同一个shard，跟其他数据就不会影响。
```



## 15 doc id手动指定和自动解析

```text
1.手动指定document id
put操作 logstash doc_id:source_timestamp_offset  md5

2.自动指定document id
长20 base64编码 GUID 分布式系统在并行生成id时是不会冲突的。 全局唯一。
```



## 16 _source元数据

```text
_source元数据：put进去的json数据，是原封不动返回来的。
可以定制：
GET /test_price/_search?_source=name
```



## 17 document全量替换、强制创建、删除

document_id存在就是全量替换，`version`增加；不存在就是创建。

内部原理：老数据标记为deleted，es数据越来越多，后台将标记为deleted状态的文件会被删除。

delete操作：

```text
DELETE /test_price/_doc/4 # 不会进行物理删除，只是标记为deleted。
```



## 18 elatsicsearch 并发冲突问题

多线程并发访问一个库存，同时购买，会导致数据不准确，出现错误。



## 19 并发控制方案：悲观锁、乐观锁

悲观锁：只有一个线程能操作数据。

elasticsearch使用的乐观锁来解决并发问题：

```text
version控制，如果2个线程同时操作数据，先执行完的线程将version+1，另外一个线程执行完会去比对version，不一致会重新查询数据进行相关操作。
```

**乐观锁、悲观锁**

```text
1.悲观锁
方便、直接加锁，对应用程序来说透明，不需要做额外操作。但是并发下只有一个线程可以获得锁，操作数据

2.乐观锁
并发能力很高
麻烦，每次更新数据都得比对版本号，可能会重复好几次。
```



## 20 elasticsearch内部如何基于乐观锁进行并发控制

修改或删除都会给数据增加版本号。

可能存在的问题：

```text
如果先修改的后到，后修改的先到，数据就可能不一致。
解决方案：
根据版本号做判断：如果后修改的先到，更新此时version，之后先修改的到了会比对版本号，如果不一致聚直接丢弃。

带上版本号去更新：
两个线程执行
PUT /test_price/_doc/7?version=3
一个先成功，另外一个执行后会出现version conflict。
```



## 22 external version

可以让用户自己维护一个version，比如说系统本身维护了一个version。

两者的区别：

```text
1._version
只有相等才能更新
2.external version
只有大于才能更新
PUT /test_price/1?version=2&version_type=external
```



## 23 partial update 

```text
PUT /test_price/_doc/7  # 需要全量发数据
{
    "name":"tutu",
    "age":18
}

POST /test_price/_doc/7/_update  # 部分发送即可，这就是partial update
{
  "doc":{
    "name":"xiaolili"
  }
}

执行过程：
1.获取document
2.将传过来的field更新到document的json中
3.将老的document标记为deleted
4.将修改后的新的document创建出来

好处：
查询、修改和写回放在shard内部，一瞬间就可以完成。可以减少并发冲突的情况。
```



## 24 groovy脚本更新document

```text
POST /test_price/_doc/6/_update
{
  "script":{
    "source":"ctx._source.age+=1"  # age可以加1
  }
}
```



## 25 partial update 乐观锁并发控制

乐观锁机制同前

retry机制：

```text
PUT /index/_doc.id/_update？retry_on_conflict=5(重试5次)
```



## 26 mget 批量查询

```text
GET /_mget
{
  "docs":[
    {
      "_index":"test_price",
      "_id":1
    },
    {
      "_index":"test_price",
      "_id":2
    }
    ]
}

# result
{
  "docs": [
    {
      "_index": "test_price",
      "_type": "_doc",
      "_id": "1",
      "_version": 1,
      "found": true,
      "_source": {
        "name": "xiaohong",
        "age": 18
      }
    },
    {
      "_index": "test_price",
      "_type": "_doc",
      "_id": "2",
      "_version": 1,
      "found": true,
      "_source": {
        "name": "xiaoming",
        "age": 20
      }
    }
  ]
}
```



## 27 bulk api

```text
Performs multiple indexing or delete operations in a single API call. This reduces overhead and can greatly increase indexing speed.

POST _bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```





## 28 elasticsearch

分布式存储文档系统

```text
1.数据量较大，es的分布式本质，可以帮助你快速进行扩容，承载大量数据。
2.数据结构灵活多变，随时可能会变化，而且数据结构之间的关系，非常复杂。如果用传统数据库，可能需要面临大量的表。
3.对数据的相关操作，较为简单。
```



## 29 document路由原理

路由算法：

```text
shard = hash(routing) % primary_shard_nums
routing默认是document id

routing可以手动指定，PUT /test/_doc/1?routing=user_id
手动指定很有用，可以保证某一类的document都放在一个shard上，对后续负载均衡、提升读取性能都很有用。
```

primary shard不能变的原因也跟路由公式有关。

```text
已有数据存储在固定shard上，现在扩primary shard，读取数据就需要重新路由，可能找不到那个shard，从而查不到数据。
```



## 30 增删改内部原理

```text
（1）客户端选择一个node发送请求过去，这个node就是coordinating node（协调节点）
（2）coordinating node，对document进行路由，将请求转发给对应的node（有primary shard）
（3）实际的node上的primary shard处理请求，然后将数据同步到replica node
（4）coordinating node，如果发现primary node和所有replica node都搞定之后，就返回响应结果给客户端
```



## 31 写一致性

```text
index.write.wait_for_active_shards可以设置
可以设置成1（default）、2、...all
其中all代表number_of_replicas+1。

5.x系列版本：
consistency=one/all(primary shard+ replica shard)/quorum((primary+numer_of_shards)/2+1)

如果shard没准备好，可以设置timeout
```



## 32 查询内部原理

```text
1、客户端发送请求到任意一个node，成为coordinate node
2、coordinate node对document进行路由，将请求转发到对应的node，此时会使用round-robin随机轮询算法，在primary shard以及其所有replica中随机选择一个，让读请求负载均衡
3、接收请求的node返回document给coordinate node
4、coordinate node返回document给客户端
5、特殊情况：document如果还在建立索引过程中，可能只有primary shard有，任何一个replica shard都没有，此时可能会导致无法读取到document，但是document完成索引建立之后，primary shard和replica shard就都有了
```



## 33 bulk api奇特的json格式

```text
POST _bulk
{ "update" : {"_id" : "1", "_type" : "_doc", "_index" : "index1", "retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"} }
{ "update" : { "_id" : "0", "_type" : "_doc", "_index" : "index1", "retry_on_conflict" : 3} }
{ "script" : { "source": "ctx._source.counter += params.param1", "lang" : "painless", "params" : {"param1" : 1}}, "upsert" : {"counter" : 1}}
{ "update" : {"_id" : "2", "_type" : "_doc", "_index" : "index1", "retry_on_conflict" : 3} }
{ "doc" : {"field" : "value"}, "doc_as_upsert" : true }
{ "update" : {"_id" : "3", "_type" : "_doc", "_index" : "index1", "_source" : true} }
{ "doc" : {"field" : "value"} }
{ "update" : {"_id" : "4", "_type" : "_doc", "_index" : "index1"} }
{ "doc" : {"field" : "value"}, "_source": true}
```

不用json格式的原因：

1.使用json格式，需要在内存中copy一份JSONArray对象，解析json对象提供路由，JSONArray对象序列化传输

这样需要消耗双倍的内存，可能触发频繁的gc。

2.更多的内存也意味着会积压其他请求的内存使用量，是其他请求性能急速下降。



采用奇特格式：

{'action':{}}

{'doc':{}}

```text
1.不用将其转成json，不会出现内存拷贝数据
2.对每两个一组的json，读取meta，进行document路由
3.直接将对应json发到node上。
```



## 34 search timeout机制揭秘

```text
{
  "took": 72, # 花费了多少毫秒
  "timed_out": false,  
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 7,  # 本次搜索返回多少结果
    "max_score": 1, # 最大的相关度分数
    "hits": [ ...] 
    }
}

timeout
默认是没有的，设置后可以在timeout内返回部分数据。时间敏感的服务需要。
```



## 35 multi search/multi index

一次性搜索多个索引

```text
GET /test*/_search
GET /test_index,test_price/_search
```



## 36 分页搜索

```text
from size
GET /test*/_search
{
  "from": 0,
  "size": 5
}

GET /test*/_search?from=0&size=5
```

**什么是deep paging**

```text
搜索特别深
场景：
总共60000条数据，每个shard上分配20000条数据，每页取10条，现在搜索第1000页，实际拿到的是10000-10010范围的数据。

3个shard都会返回10010条数据，然后合并排序，取出第10000页的数据。

这种deep paging操作，耗费网络带宽、内存、cpu，所以应当尽量避免这种操作
```



## 37 query string search 以及_all metadata原理揭秘

```text
GET /test_price/_search?q=name:tutu # 包含name:tutu
GET /test_price/_search?q=+name:tutu # 包含name:tutu
GET /test_price/_search?q=-name:tutu  # 不包含name:tutu
```

_all metadata

```text
GET /test_price/_search?q=test

可以直接搜索所有field，任何一个field包含指定的关键字都能被搜索出来

原理：
es中的_all元数据在建立索引时，插入一条document，es会将document中的field用字符串形式串起来，编变成一个长字符串，作为all_field的值，同时建立索引。

生成环境不使用
```



## 38 mapping

包含了每个field对应的数据类型，以及如何分词等设置



## 40 倒排索引核心原理

```text
normalization 建立倒排索引的时候，会执行一个操作，就是对拆分出的各个单词进行相应的处理，以提升后面搜索到的时候能够搜索到相关联文档的概率。
```



## 41 分词器

```text
# 概念
切分词语，normalization（标准化，时态转换、同义词转换、大小写转换）

给你一段句子，然后将这段句子拆分成一个一个的单个的单词，同时对每个单词进行normalization（时态转换，单复数转换），分词器。recall，召回率：搜索的时候，增加能够搜索到的结果的数量

character filter：在一段文本进行分词之前，先进行预处理，比如说最常见的就是，过滤html标签（<span>hello<span> --> hello），& --> and（I&you --> I and you）
tokenizer：分词， 例如：hello you and me --> hello, you, and, me
token filter：lowercase，stop word 例如：dogs --> dog，liked --> like，Tom --> tom，a/the/an --> 干掉，mother --> mom，small --> little

一个分词器，很重要，将一段文本进行各种处理，最后处理好的结果才会拿去建立倒排索引

# 内置分词器介绍
Set the shape to semi-transparent by calling set_trans(5)

standard analyzer：set, the, shape, to, semi, transparent, by, calling, set_trans, 5（默认的是standard）

simple analyzer：set, the, shape, to, semi, transparent, by, calling, set, trans

whitespace analyzer：Set, the, shape, to, semi-transparent, by, calling, set_trans(5)

language analyzer（特定的语言的分词器，比如说，english，英语分词器）：set, shape, semi, transpar, 
call, set_tran, 5
```



## 42 mapping

```text
1.GET /_search?q=2017

搜索的是_all field，document所有的field都会拼接成一个大串，进行分词

2017-01-02 my second article this is my second article in this website 11400

		doc1		doc2		doc3
2017	*			*			*
01		* 		
02					*
03								*

_all，2017，自然会搜索到3个docuemnt

2.GET /_search?q=2017-01-01

_all，2017-01-01，query string会用跟建立倒排索引一样的分词器去进行分词

2017
01
01

3.GET /_search?q=post_date:2017-01-01

date，会作为exact value去建立索引

		doc1		doc2		doc3
2017-01-01	*		
2017-01-02			* 		
2017-01-03					*

post_date:2017-01-01，2017-01-01，doc1一条document

4.GET /_search?q=post_date:2017，这个在这里不讲解，因为是es 5.2以后做的一个优化
```



## 49 filter与query深入对比解密：相关度、性能

```text
filter仅仅是过滤数据，内置cache可以使用filter，性能更好
query会计算每个document相对于搜索条件的相关度，并按相关度进行排序。不能在cache中使用，性能较差。
```



## 54 针对string排序

```text
对一个string field进行排序，结果往往不准确，因为分词后会是多个单词
处理方案:
将一个string field建立两次索引，一个分词用来建立索引，一个不分词用于排序。
# 在mapping里面设置 5.x系列
"title":{
    "type":"text",
    "fields":{
        "raw": {
            "type":"string",
            "index":"not_analyzed",  # 6.x取消了 直接将字段处理成keyword形式
            "fielddata":true
        }
    }
}
# 5.x系列搜索语句
GET /test/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
  	“title.raw”:{
        "order":"desc"
  	}
  	]
}

# 6.x系列搜索语句
GET /test_price/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "name.keyword": {
        "order": "desc"
      }
    }
  ]
}
```



## 55 相关度评分TF&IDF算法独家揭秘

```text
使用的是term frequency/inverse document frequency算法

1.Term frequency：搜索文本中的各个词条在field文本中出现了多少次，出现次数越多，就越相关

搜索请求：hello world
doc1：hello you, and world is very good
doc2：hello, how are you
doc1更相关

tf(t in d) = sqrt(frequency) ， 即 (某个词t在文档d中出现的次数)的平方跟

2.Inverse document frequency：搜索文本中的各个词条在整个索引的所有文档中出现了多少次，出现的次数越多，就越不相关

搜索请求：hello world
doc1：hello, today is very good
doc2：hi world, how are you

比如说，在index中有1万条document，hello这个单词在所有的document中，一共出现了1000次；world这个单词在所有的document中，一共出现了100次
doc2更相关

idf(t) = 1 + log ( numDocs / (docFreq + 1)) 
1 + log ( 索引中的文档总数 / (包含该词的文档数 + 1))

3.Field-length norm（字段长度归一化）：field长度，field越长，相关度越弱

搜索请求：hello world

doc1：{ "title": "hello article", "content": "babaaba 1万个单词" }
doc2：{ "title": "my article", "content": "blablabala 1万个单词，hi world" }

hello world在整个index中出现的次数是一样多的

doc1更相关，title field更短

norm(d) = 1 / sqrt(numTerms)    即： 1/词出现次数的平方根
```

elasticsearch

```text
idf: log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5))
tfNorm:(freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength))
其中：termFreq＝1，k1=1.2, b=0.75

_score = idf * tfNorm
```



## 56 doc values

正排索引 过滤排序

```text
doc1: {"name":"jack","age":11}
doc2: {"name":"lili","age":12}

document  name  age
doc1      jack   11
doc2      lili   12
```



## 57 query phase

  ```text
1. 搜索请求发送到某一个coordinate node，构建一个priority queue，长度以paging操作from和size为准，默认为10

2.coordinate node将请求转发到所以shard，每个shard本地搜索，并构建一个本地的priority queue

3.各个shard将自己的priority queue返回给coordinate node，并构建一个全局的priority queue.
  ```



## 58 fetch phase

Query phase结束后获取的是一堆doc id
之后进行fetch phase

```text
1.coordinate node构建完priority queue之后，就发送mget请求去所有的shard上获取对应的document

2.各个shard将document返回给coordinate node

3.coordinate node将合并后的document结果返回给client客户端。

# 不加from、size 默认是获取前10条 按照_score排序。
```



## 59 bouncing results

两个documen排序，field值相同，不同的shard上，可能排序不同；每次请求轮询达到不同的shard上，每个页面看到的搜索结果排序都不一样。

这就是跳跃问题。

解决方案：

将preference设置成一个随机的字符串，比如说user id，让每个user每次搜索的时候，都是用同一个shard上的数据。这里利用的是类似session 的概念：

```text
 If two searches both give the same custom string value for their preference and the underlying cluster state does not change then the same ordering of shards will be used for the searches. 
 
This does not guarantee that the exact same shards will be used each time: the cluster state, and therefore the selected shards, may change for a number of reasons including shard relocations and shard failures, and nodes may sometimes reject searches causing fallbacks to alternative nodes. 
 
However, in practice the ordering of shards tends to remain stable for long periods of time. A good candidate for a custom preference value is something like the web session id or the user name.
```



## 60 scroll滚动查询大量数据

使用scroll滚动搜索，可以一批一批的搜索数据。

scroll第一次搜索时候，会保存一个当时的视图快照，后续只会基于旧的视图快照提供数据搜索，此时即使数据变更，是不会让用户知道的。



```text
# 第一次搜索
GET test_price/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "sort": [ "_doc"],
  "size": 4
}

# 后续搜索指定scroll_id
GET _search/scroll
{
  "scroll":"1m",
  "scroll_id":"DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAB1iFldRamdKbTNNU09xdlRtOEFjOTZ2cUEAAAAAAAAfyRY5SjYxbEtoOVRudU8wUGhwYmg4b2VnAAAAAAAAH8oWOUo2MWxLaDlUbnVPMFBocGJoOG9lZwAAAAAAAB1jFldRamdKbTNNU09xdlRtOEFjOTZ2cUEAAAAAAAAdZBZXUWpnSm0zTVNPcXZUbThBYzk2dnFB"
}
```



## 63 定制自己的分词器

```text
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "&_to_and": {
          "type": "mapping",
          "mappings": ["&=> and"]
        }
      },
      "filter": {
        "my_stopwords": {
          "type": "stop",
          "stopwords": ["the", "a"]
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip", "&_to_and"],
          "tokenizer": "standard",
          "filter": ["lowercase", "my_stopwords"]
        }
      }
    }
  }
}

GET /my_index/_analyze
{
  "text": "tom&jerry are a friend in the house, <a>, HAHA!!",
  "analyzer": "my_analyzer"
  
  
```



## 67 倒排索引结构

```text
1.包含这个关键词的document list
2.包含这个关键词的所有document的数量，IDF(inverse document frequency)
3.这个关键词在每个document中出现的次数，TF(term frequency)
4.这个关键词在这个document中的次序
5.每个document的长度：length norm
6.包含这个关键词的所以偶document的平均长度。
```

倒排索引不可变的好处：

```text
1.不需要锁，提升并发能力，避免锁的问题
2.数据不变，一直保存在os cache中，只要cache内存
3.filter cache一直驻留在内存，因为数据不变
4.可以压缩，节省cpu和io开销。
```

坏处：

每次都要重新构建整个索引。



## 72 merge操作

```text
1.选择一些大小相似的segment merge成一个大的segment
2.直接将新的segment flush到disk上
3.写一个新的commit point，包括了新的segment，并且排除旧的那些segment
4.将新的segment打开供搜索
5.将旧的segment删除。
```



## elasticsearch扩容方案

一般采用水平扩容方案。

按照官方文档来操作：(每个node都是master、data节点)

```text
1.创建一个es实例
2.配置elasticsearch.yml文件，在里面加入：
cluster.name: "logging-prod"  # 集群名称
3.启动新创建的es实例，节点自动加入集群。
4.基于shard的rebalance实现shard迁移。
```



## elasticsearch项目经验准备

1.改变elasticsearch默认分词的情况:

```text
场景:日志采集后发现某个字段尾部带有下划线，搜索的时候取非下划线部分找不到，说明使用默认分词器基本不行。

1.设置自定义的分词器，创建新索引：
PUT /analyzer-new
{
  "settings":{
      "analysis" : {
         "analyzer" : { 
            "comment_analyzer" : { 
               "type": "custom",   //自定义分词器
               "tokenizer" : "standard",
               "filter" : [ "underscore", "lowercase"] 
            }
         },
         "filter" : {
            "underscore" : { 
               "type" : "pattern_capture",  //使用了pattern_capture
               "preserve_original" : true, 
               "patterns" : [ 
                  "([^_]+)" // 正则匹配再去搜索
               ] 
            }
         }
      }
  },
  "mappings": {
      "_doc": {
        "properties": {
            "word": {
                "type": "text",
                "analyzer": "comment_analyzer", 
                "fields": {
                    "keyword": {
                        "type": "keyword",
                        "ignore_above": 256
                    }
                }
            },
        	"title": {
        	"type":"text"
     }
  }
  }
  }
 }
 
 2.将旧数据更新到新的索引上
 POST /_reindex?pretty
{
  "source": {
    "index": "analyzer-test"
  },
  "dest": {
    "index": "analyzer-new"
  }
}

3.修改alias指向新的index
# 如果旧索引没有alias
POST /_aliases?pretty
{
    "actions" : [
        {"remove_index": {"index": "analyzer-test"}},
        { "add" : { "index" : "analyzer-new", "alias" : "analyzer-test" } }
    ]
}
# 如果有的话可以直接修改，指向新索引。


旧索引如果一直在写入，可以同时将索引数据写入新、老索引，先reindex此刻timestamp前的数据，等数据搬迁完，再reindex剩下的数据。
```

