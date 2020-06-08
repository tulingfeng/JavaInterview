## 1.spark部署模式

集群部署

```text
eigen部署配置：
yarn-cluster，需要配置hdfs
Resource Manager1台(4core8G) NodeMangaer当初14台(16core32G)
```

各部分作用：

![image-20200327171919512](/Users/tulingfeng/Library/Application Support/typora-user-images/image-20200327171919512.png)

**执行流程**

![image-20200327172504808](/Users/tulingfeng/Library/Application Support/typora-user-images/image-20200327172504808.png)



## 2.RDD

弹性分布式数据集

```text
1.集合创建
sc.parallelize()
sc.makdRDD()
2.由外部存储系统的数据集创建
sc.textFile(inputPath)
3.其他rdd创建
```

 

## 3.算子

1.map和mapPartitions

map是对rdd中的每一个元素进行操作。

mapPartitions则是对rdd中的每个分区的迭代器进行操作。（可能会导致OOM）



2.wordcount

```python
# 统计一个字符表在一篇文章中出现的字频
from pyspark import SparkContext
from operator import add
import time

sc = SparkContext('local', 'test')

rdd1 = sc.textFile('./data/data.txt') \
      .flatMap(lambda x:x.split(" ")) \
      .map(lambda x:(x,1)) \
      .reduceByKey(add)

rdd2 = sc.parallelize(['crazy','is','smart'])
rdd2_set = sc.parallelize(['crazy','is','smart']).map(lambda x:(x,1))

# cogroup形式
start1 = time.time()
print(rdd1.cogroup(rdd2_set).filter(lambda x:x[1][0] and x[1][1]).map(lambda x:(x[0], list(x[1][0])[0])).collect())
print('time costs:',time.time()-start1)
# broadcast形式
start2 = time.time()
bcast = sc.broadcast(rdd2.collect())
print(rdd1.filter(lambda fields: fields[0] in bcast.value).collect())
print('time costs:',time.time()-start2)
# join形式
start3 = time.time()
print(rdd1.join(rdd2_set).map(lambda x:(x[0],x[1][0])).collect())
print('time costs:',time.time()-start3)
```



3.sum

```python
from pyspark import SparkContext

sc = SparkContext('local', 'test')

rdd = sc.parallelize(range(100))

# sum方法
print(rdd.sum())

# fold方法
print(rdd.fold(0, (lambda x, y: x + y)))
```



4.average

```python
from pyspark import SparkContext
from pyspark.sql import SparkSession
from pyspark.sql.functions import *


sc = SparkContext('local','test')
nums = sc.parallelize([1, 2, 2, 3, 4, 5, 6, 7, 8, 20])

# mean方法
print('average:',nums.mean())


# fold方法
sumAndCount = nums.map(lambda x: (x, 1)).fold((0, 0), (lambda x, y: (x[0] + y[0], x[1] + y[1])))
print('avrage:',float(sumAndCount[0]) / float(sumAndCount[1]))

# df agg
spark = SparkSession.builder \
    .master("local") \
    .appName("Word Count") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()

rdd = nums.map(lambda x:(x,1))
df = spark.createDataFrame(rdd,["age","id"]).agg({"age":"avg"}).collect()
print(df)
```



5.broadcast

```python
# Broadcast a read-only variable to the cluster, returning a L{Broadcast<pyspark.broadcast.Broadcast>} object for reading it in distributed functions. The variable will be sent to each cluster only once.

from pyspark import SparkContext

sc = SparkContext('local', 'test')
b = sc.broadcast([1, 2, 3, 4, 5])

print(sc.parallelize([0, 0]).flatMap(lambda x: b.value).collect())
```



6.glom

```python
# Return an RDD created by coalescing all elements within each partition into a list.
 
from pyspark import SparkContext

sc = SparkContext("local","test")

rdd = sc.parallelize([1, 2, 3, 4,5,6,7], 2).glom()

print(rdd.collect())

# output
[[1, 2, 3], [4, 5, 6, 7]]
```



7.groupBy

```python
# Return an RDD of grouped items.

from pyspark import SparkContext

sc = SparkContext("local","test")

rdd = sc.parallelize([1, 1, 2, 3, 5, 8])

result = rdd.groupBy(lambda x:x%2).collect()

print(sorted([(x, sorted(y)) for (x, y) in result]))

# output
[(0, [2, 8]), (1, [1, 1, 3, 5])]
```



8.distinct

```python
# Return a new RDD containing the distinct elements in this RDD.

from pyspark import SparkContext

sc = SparkContext("local","test")

print(sorted(sc.parallelize([1, 1, 1,2,3,2, 3]).distinct().collect()))

# output
[1, 2, 3]
```



9.coalesce

```python
# Return a new RDD that is reduced into numPartitions partitions.
from pyspark import SparkContext

sc = SparkContext("local","test")

print(sc.parallelize([1, 2, 3, 4, 5], 3).glom().collect())
# [[1], [2, 3], [4, 5]]
print(sc.parallelize([1, 2, 3, 4, 5], 3).coalesce(1).glom().collect())
# [[1, 2, 3, 4, 5]]
```



10.checkpoint

```python
from pyspark import SparkContext
from pyspark.sql.functions import *


sc = SparkContext('local','test')
sc.setCheckpointDir('./data/ck')  # 需要先设置chekpointdir
nums = sc.parallelize([1, 2, 2, 3, 4, 5, 6, 7, 8, 20])

nums.checkpoint()  # checkpoint属于懒加载，先

print(nums.sum())
```



## 4.任务划分

```text
1.Application:初始化一个SparkContext即生成一个Application
2.Job:一个Action算子就会生成一个job
3.Stage:根据RDD之间的依赖关系的不同将Job划分成不同的Stage，遇到一个宽依赖则划分一个Stage
4.Task:Stage是一个TaskSet，将Stage划分的记过发送到不同的Executor执行即为一个Task

Application --> Job --> Stage --> Task 每层都是1对n的关系。
```



## 5.三大数据结构

1.RDD：分布式数据集



2.广播变量：分布式只读共享变量

广播变量的做法：就是不把副本变量分发到每个 Task 中，而是将其分发到每个 Executor，Executor 中的所有 Task 共享一个副本变量。

```python
from pyspark import SparkContext

sc = SparkContext('local', 'test')
b = sc.broadcast([1, 2, 3, 4, 5])

print(sc.parallelize([0, 0]).flatMap(lambda x: b.value).collect())
```



3.累加器：分布式只写变量

```python
from pyspark import SparkContext

sc = SparkContext("local","test")

nums = sc.parallelize(range(10))

count = 0
def f(x):
    global count
    count += x

rdd = nums.foreach(f)
print('count:',count)
# output  0 
# spark启动后会将count变量拷贝成副本变量，副本变量与函数一起形成闭包，序列化并发送给每一个执行者。
# 因此，当在 foreach 函数中引用 count 时，它将不再是 Driver 节点上的 count，而是闭包中的副本 count，默认情况下，副本 count 更新后的值不会回传到 Driver，所以 count 的最终值仍然为零。


a = sc.accumulator(0)
def f1(x):
    global a
    a += x
rdd_accu = nums.foreach(f1)

print('a:',a.value)
# output 45
# 解决上述闭包问题是引入累加器，将每个副本变量的最终值传回 Driver，由 Driver 聚合后得到最终值，并更新原始变量。
```



## 6.spark总结

**弹性**

```text
1.血缘(依赖关系)
Spark可以通过特殊的处理方案简化的依赖关系
2.计算
Spark的计算是基于内存的，所以性能特别高，可以和磁盘灵活切换
3.分区
Spark在创建默认分区后，可以通过指定的算子来改变分区数量
4.容错
Spark在执行计算时，如果发生错误，可以进行容错重试处理
```

**Spark中的数量**

```text
1.Executor：可以通过提交应用的参数进行设定
2.Partition：默认情况下，读取文件采用的是Hadoop的切片规则，如果读取内存中的数据，可以根据特定的算法进行设定，可以通过其他算子进行改变。
多个阶段的场合，下一个阶段的分区数量取决于上一个阶段最后RDD的分区数，但是可以在相应的算子中进行修改。
3.Stage：1(ResultStage) + Shuffle依赖的数量(ShuffleMapStage)划分阶段的目的就是为了任务执行的等待，因为Shuffle的过程需要
4.Task：原则上一个分区就是一个任务，但是实际应用中可以动态调整
```



## 7.spark streaming vs structured streaming

[spark streaming](<https://zhuanlan.zhihu.com/p/47838090>)

[structured streaming](<https://zhuanlan.zhihu.com/p/51883927>)

### 7.1 spark streaming

![image-20200328225548263](/Users/tulingfeng/Library/Application Support/typora-user-images/image-20200328225548263.png)

分布式计算主要分两种：批（batch）处理和流式（streaming）计算，流式计算的主要优势在于其时效性和低延迟。而大规模流式计算系统设计的两个主要问题是错误处理和 straggler 处理。

straggler指的是分布式系统某一个成员/成分运行滞后于其他组成部分，比如某个 task 节点的运行时间要明显长于其他节点。

一般流式计算（storm）等对错误处理方案：

```text
1.replication: 每个operator节点设置一个replication，耗费一倍资源。
2.replay: 上游replay，耗费一定时间。
```

并且都没有针对straggler的处理。

**spark streaming对错误的处理**

Spark Streaming 的模式是 discretized streams (D-Streams)，这种模式不存在一直运行的 operator，而是将每一个时间间隔的数据通过一系列无状态、确定性（deterministic）的批处理来处理。

对错误处理和straggler处理的方式分别是：

```text
1.错误处理：
使用的是parallel recovery 当一个节点失败之后，我们通过集群的其他节点来一起快速重建出失败节点的 RDD 数据。具体做法是通过partition来处理的。
2.straggler处理：
 straggler 的处理因为我们可以获取到一个批处理 task 的运行时间，所以我们可以通过推测 task 的运行时间判断是不是 straggler。
```

**spark streaming快的原因**

DStream 使用弹性分布式数据集（Resilient Distributed Datasets），也就是 RDD，来进行批处理（注：RDD 可以将数据保存到内存中，然后通过 RDD 之间的依赖关系快速计算）。这个过程一般是亚秒级的，对于大部分场景都是可以满足的。

**数据一致性**

每个时间 interval 对应的 RDDs 都是容错不可变且计算确定性的，所以 DStream 是满足 exactly-once 语义的。



**不足**

1.**使用 Processing Time 而不是 Event Time**

2.**Complex, low-level api**。这点比较好理解，DStream （Spark Streaming 的数据模型）提供的 API 类似 RDD 的 API 的。

3.**reason about end-to-end application**。这里的 end-to-end 指的是直接 input 到 out，比如 Kafka 接入 Spark Streaming 然后再导出到 HDFS 中。DStream 只能保证自己的一致性语义是 exactly-once 的，而 input 接入 Spark Streaming 和 Spark Straming 输出到外部存储的语义往往需要用户自己来保证。而这个语义保证写起来也是非常有挑战性，比如为了保证 output 的语义是 exactly-once 语义需要 output 的存储系统具有幂等的特性，或者支持事务性写入，这个对于开发者来说都不是一件容易的事情。

4.**批流代码不统一**。尽管批流本是两套系统，但是这两套系统统一起来确实很有必要，我们有时候确实需要将我们的流处理逻辑运行到批数据上面。关于这一点，最早在 2014 年 Google 提出 Dataflow 计算服务的时候就批判了 streaming/batch 这种叫法，而是提出了 unbounded/bounded data 的说法。DStream 尽管是对 RDD 的封装，但是我们要将 DStream 代码完全转换成 RDD 还是有一点工作量的，更何况现在 Spark 的批处理都用 DataSet/DataFrame API 了。



### 7.2 structured streaming

**structured streaming优势**

- **Incremental query model**: Structured Streaming 将会在新增的流式数据上不断执行增量查询，同时代码的写法和批处理 API （基于 Dataframe 和 Dataset API）完全一样，而且这些 API 非常的简单。
- **Support for end-to-end application**: Structured Streaming 和内置的 connector 使的 end-to-end 程序写起来非常的简单，而且 "correct by default"。数据源和 sink 满足 "exactly-once" 语义，这样我们就可以在此基础上更好地和外部系统集成。
- **复用 Spark SQL 执行引擎**：我们知道 Spark SQL 执行引擎做了非常多的优化工作，比如执行计划优化、codegen、内存管理等。这也是 Structured Streaming 取得高性能和高吞吐的一个原因。

![preview](https://pic2.zhimg.com/v2-8467b7d62beb4c6353e9504b0b616e81_r.jpg)

**Structured Streaming 核心设计**

- **Input and Output**: Structured Streaming 内置了很多 connector 来保证 input 数据源和 output sink 保证 exactly-once 语义。而实现 exactly-once 语义的前提是：

- - Input 数据源必须是可以 replay 的，比如 Kafka，这样节点 crash 的时候就可以重新读取 input 数据。常见的数据源包括 Amazon Kinesis, Apache Kafka 和文件系统。
  - Output sink 必须要支持写入是幂等的。这个很好理解，如果 output 不支持幂等写入，那么一致性语义就是 at-least-once 了。另外对于某些 sink, Structured Streaming 还提供了原子写入来保证 exactly-once 语义。

- **API**: Structured Streaming 代码编写完全复用 Spark SQL 的 batch API，也就是对一个或者多个 stream 或者 table 进行 query。query 的结果是 result table，可以以多种不同的模式（append, update, complete）输出到外部存储中。另外，Structured Streaming 还提供了一些 Streaming 处理特有的 API：Trigger, watermark, stateful operator。

- **Execution**: 复用 Spark SQL 的执行引擎。Structured Streaming 默认使用类似 Spark Streaming 的 micro-batch 模式，有很多好处，比如动态负载均衡、再扩展、错误恢复以及 straggler （straggler 指的是哪些执行明显慢于其他 task 的 task）重试。除了 micro-batch 模式，Structured Streaming 还提供了基于传统的 long-running operator 的 continuous 处理模式。

- **Operational Features**: 利用 wal 和状态存储，开发者可以做到集中形式的 rollback 和错误恢复。还有一些其他 Operational 上的 feature，这里就不细说了。

**编程模型**

Structured Streaming 将流式数据当成一个不断增长的 table，然后使用和批处理同一套 API，都是基于 DataSet/DataFrame 的。

![image-20200328223603089](/Users/tulingfeng/Library/Application Support/typora-user-images/image-20200328223603089.png)



组成部分：

- **Input Unbounded Table**: 流式数据的抽象表示
- **Query**: 对 input table 的增量式查询
- **Result Table**: Query 产生的结果表
- **Output**: Result Table 的输出

```python
# Create DataFrame representing the stream of input lines from connection to localhost:9999
lines = spark \
    .readStream \
    .format("socket") \
    .option("host", "localhost") \
    .option("port", 9999) \
    .load()

# Split the lines into words
words = lines.select(
   explode(
       split(lines.value, " ")
   ).alias("word")
)

# Generate running word count
wordCounts = words.groupBy("word").count()

 # Start running the query that prints the running counts to the console
query = wordCounts \
    .writeStream \
    .outputMode("complete") \
    .format("console") \
    .start()
```

**执行逻辑**

![preview](https://pic2.zhimg.com/v2-9b2b6bab9683c91de847022e5c78c0e9_r.jpg)



把流式数据当成一张不断增长的 table，也就是图中的 Unbounded table of all input。然后每秒 trigger 一次，在 trigger 的时候将 query 应用到 input table 中新增的数据上，有时候还需要和之前的静态数据一起组合成结果。query 产生的结果成为 Result Table，我们可以选择将 Result Table 输出到外部存储。输出模式有三种：

- **Complete mode**: Result Table 全量输出
- **Append mode (default)**: 只有 Result Table 中新增的行才会被输出，所谓新增是指自上一次 trigger 的时候。因为只是输出新增的行，所以如果老数据有改动就不适合使用这种模式。
- **Update mode**: 只要更新的 Row 都会被输出，相当于 Append mode 的加强版。



**Continuous Processing Mode**

Spark streaming使用的是micro-batch 模式。不能算真正的流处理。

structured streaming使用的是：

continuous mode 这种处理模式只要一有数据可用就会进行处理，如下图所示。epoch 是 input 中数据被发送给 operator 处理的最小单位，在处理过程中，epoch 的 offset 会被记录到 wal 中。另外 continuous 模式下的 snapshot 存储使用的一致性算法是 Chandy-Lamport 算法。

![preview](https://pic1.zhimg.com/v2-dd8ea8962b30ef13dfb679d6a3a89350_r.jpg)

这种模式相比与 micro-batch 模式缺点和优点都很明显。

- 缺点是不容易做扩展
- 优点是延迟更低

优点展示：![image-20200328234918409](/Users/tulingfeng/Library/Application Support/typora-user-images/image-20200328234918409.png)



![preview](https://pic4.zhimg.com/v2-56d7da66f9d62cc38fcb8d5dc80267bb_r.jpg)

**一致性语义**

**micro-batch** 模式可以提供 end-to-end 的 exactly-once 语义。原因是因为在 input 端和 output 端都做了很多工作来进行保证，比如 input 端 replayable + wal，output 端写入幂等。

**continuous mode** 只能提供 at-least-once 语义。