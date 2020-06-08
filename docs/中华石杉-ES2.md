## 02 filter term

```text
POST /forum/_doc/_bulk
{ "index": { "_id": 1 }}
{ "articleID" : "XHDK-A-1293-#fJ3", "userID" : 1, "hidden": false, "postDate": "2017-01-01" }
{ "index": { "_id": 2 }}
{ "articleID" : "KDKE-B-9947-#kL5", "userID" : 1, "hidden": false, "postDate": "2017-01-02" }
{ "index": { "_id": 3 }}
{ "articleID" : "JODL-X-1937-#pV7", "userID" : 2, "hidden": false, "postDate": "2017-01-01" }
{ "index": { "_id": 4 }}
{ "articleID" : "QQPX-R-3956-#aD8", "userID" : 2, "hidden": true, "postDate": "2017-01-02" }

GET /forum/_doc/_search
{
    "query" : {
        "constant_score" : { 
            "filter" : {
                "term" : { 
                    "articleID" : "XHDK-A-1293-#fJ3"
                }
            }
        }
    }
}
搜不出来
但是用articleID.keyword可以搜出来。
```



## 03 filter执行原理深度解析

```text
1）在倒排索引中查找搜索串，获取document list

date来举例

word		doc1		doc2		doc3

2017-01-01	*		*
2017-02-02			*		*
2017-03-03	*		*		*

filter：2017-02-02

到倒排索引中一找，发现2017-02-02对应的document list是doc2,doc3

（2）为每个在倒排索引中搜索到的结果，构建一个bitset，[0,1,1]

非常重要

使用找到的doc list，构建一个bitset，就是一个二进制的数组，数组每个元素都是0或1，用来标识一个doc对一个filter条件是否匹配，如果匹配就是1，不匹配就是0

[0, 1, 1]

doc1：不匹配这个filter的
doc2和do3：是匹配这个filter的

尽可能用简单的数据结构去实现复杂的功能，可以节省内存空间，提升性能

（3）遍历每个过滤条件对应的bitset，优先从最稀疏的开始搜索，查找满足所有条件的document

后面会讲解，一次性其实可以在一个search请求中，发出多个filter条件，每个filter条件都会对应一个bitset
遍历每个filter条件对应的bitset，先从最稀疏的开始遍历

[0, 0, 0, 1, 0, 0]：比较稀疏
[0, 1, 0, 1, 0, 1]

先遍历比较稀疏的bitset，就可以先过滤掉尽可能多的数据

遍历所有的bitset，找到匹配所有filter条件的doc

请求：filter，postDate=2017-01-01，userID=1

postDate: [0, 0, 1, 1, 0, 0]
userID:   [0, 1, 0, 1, 0, 1]

遍历完两个bitset之后，找到的匹配所有条件的doc，就是doc4

就可以将document作为结果返回给client了

（4）caching bitset，跟踪query，在最近256个query中超过一定次数的过滤条件，缓存其bitset。对于小segment（<1000，或<3%），不缓存bitset。

比如postDate=2017-01-01，[0, 0, 1, 1, 0, 0]，可以缓存在内存中，这样下次如果再有这个条件过来的时候，就不用重新扫描倒排索引，反复生成bitset，可以大幅度提升性能。

在最近的256个filter中，有某个filter超过了一定的次数，次数不固定，就会自动缓存这个filter对应的bitset

segment（上半季），filter针对小segment获取到的结果，可以不缓存，segment记录数<1000，或者segment大小<index总大小的3%

segment数据量很小，此时哪怕是扫描也很快；segment会在后台自动合并，小segment很快就会跟其他小segment合并成大segment，此时就缓存也没有什么意义，segment很快就消失了

针对一个小segment的bitset，[0, 0, 1, 0]

filter比query的好处就在于会caching，但是之前不知道caching的是什么东西，实际上并不是一个filter返回的完整的doc list数据结果。而是filter bitset缓存起来。下次不用扫描倒排索引了。

（5）filter大部分情况下来说，在query之前执行，先尽量过滤掉尽可能多的数据

query：是会计算doc对搜索条件的relevance score，还会根据这个score去排序
filter：只是简单过滤出想要的数据，不计算relevance score，也不排序

（6）如果document有新增或修改，那么cached bitset会被自动更新

postDate=2017-01-01，[0, 0, 1, 0]
document，id=5，postDate=2017-01-01，会自动更新到postDate=2017-01-01这个filter的bitset中，全自动，缓存会自动更新。postDate=2017-01-01的bitset，[0, 0, 1, 0, 1]
document，id=1，postDate=2016-12-30，修改为postDate-2017-01-01，此时也会自动更新bitset，[1, 0, 1, 0, 1]

（7）以后只要是有相同的filter条件的，会直接来使用这个过滤条件对应的cached bitset
```



## ngram

```text
The ngram tokenizer first breaks text down into words whenever it encounters one of a list of specified characters, then it emits N-grams of each word of the specified length.

N-grams are like a sliding window that moves across the word - a continuous sequence of characters of the specified length. 


滑动窗口

POST _analyze
{
  "tokenizer": "ngram",
  "text": "Quick Fox"
}

[ Q, Qu, u, ui, i, ic, c, ck, k, "k ", " ", " F", F, Fo, o, ox, x ]

参数：
min_gram max_gram
例如：
hello world

min ngram = 1
max ngram = 3

h
he
hel
w
wo
wor

测试：
PUT /my_index
{
    "settings": {
        "analysis": {
            "filter": {
                "autocomplete_filter": { 
                    "type":     "edge_ngram",
                    "min_gram": 1,
                    "max_gram": 20
                }
            },
            "analyzer": {
                "autocomplete": {
                    "type":      "custom",
                    "tokenizer": "standard",
                    "filter": [
                        "lowercase",
                        "autocomplete_filter" 
                    ]
                }
            }
        }
    }
}

PUT /my_index/_mapping
{
  "properties": {
      "title": {
          "type":     "string",
          "analyzer": "autocomplete",
          "search_analyzer": "standard"  # 搜索还是用标准分词器
      }
  }
}

GET /my_index/_search 
{
  "query": {
    "match": {
      "title": "hello w"
    }
  }
}

# 如果用match的话，hello world/hello win/hello dog都可以匹配，全文检索，hello dog分数比较低。

推荐使用match_phrase，要求每个term都有，而且position刚好靠着1位，符合我们的期望。
```



 ## 26  TF/IDF

```text
单个term在doc中的分数

query: hello world --> doc.content
doc1: java is my favourite programming language, hello world !!!
doc2: hello java, you are very good, oh hello world!!!

hello对doc1的评分

TF: term frequency 

找到hello在doc1中出现了几次，1次，会根据出现的次数给个分数
一个term在一个doc中，出现的次数越多，那么最后给的相关度评分就会越高

IDF：inversed document frequency

找到hello在所有的doc中出现的次数，3次
一个term在所有的doc中，出现的次数越多，那么最后给的相关度评分就会越低

length norm

hello搜索的那个field的长度，field长度越长，给的相关度评分越低; field长度越短，给的相关度评分越高

最后，会将hello这个term，对doc1的分数，综合TF，IDF，length norm，计算出来一个综合性的分数

hello world --> doc1 --> hello对doc1的分数，world对doc1的分数 --> 但是最后hello world query要对doc1有一个总的分数 --> vector space model
```

Vector space  model

```text
 多个term对一个doc的总分数
 
 hello world  --> es会根据hello world在所有doc中的评分情况，计算出一个query vector
 hello这个term 给的基于所有doc的一个评分是2
 world这个term 给的基于所有doc的一个评分是5
 
 [2,5] query vecor
 
 则对于doc vector
 doc1： 包含hello --> [2,0]
 doc2:  包含world --> [0,5]
 doc3:  包含hello,world --> [2,5]
 
 每个doc vector计算query vector的弧度，最后基于这个弧度给出一个doc相对于query中多个term的总分数。
 弧度越大，分数越低；弧度越小，分数越高。
```



## 27 深入揭秘lucene的相关度分数算法

 ```text
score(q,d)  =  
            queryNorm(q)  
          · coord(q,d)    
          · ∑ (           
                tf(t in d)   
              · idf(t)2      
              · t.getBoost() 
              · norm(t,d)    
            ) (t in q) 

参数说明：
1.queryNorm(q) is the query normalization factor 
queryNorm，是用来让一个doc的分数处于一个合理的区间内，不要太离谱，举个例子，一个doc分数是10000，一个doc分数是0.1，肯定不合适。

queryNorm = 1 / √sumOfSquaredWeight
sumOfSquaredWeights = 所有term的IDF分数之和。

2.coord(q,d) is the coordination factor
对更加匹配的doc，进行一些分数上的成倍奖励

奖励那些匹配更多字符的doc更多的分数

Document 1 with hello → score: 1.5
Document 2 with hello world → score: 3.0
Document 3 with hello world java → score: 4.5

Document 1 with hello → score: 1.5 * 1 / 3 = 0.5
Document 2 with hello world → score: 3.0 * 2 / 3 = 2.0
Document 3 with hello world java → score: 4.5 * 3 / 3 = 4.5

3.tf/idf评分参照上面

4.t.getboost()
t.getBoost() is the boost that has been applied to the query
 ```



## 28 四种常见的相关度评分优化算法

```text
1.改变搜索结构来改变权重
GET /forum/_search 
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "content": "java"
          }
        },
        {
          "match": {
            "content": "spark"
          }
        },
        {
          "bool": {  # 层级更深
            "should": [
              {
                "match": {
                  "content": "solution"
                }
              },
              {
                "match": {
                  "content": "beginner"
                }
              }
            ]
          }
        }
      ]
    }
  }
}

2.采用boost(提升)
POST _search
{
    "query": {
        "match" : {
            "title": {
                "query": "quick brown fox",
                "boost": 2
            }
        }
    }
}

如果想降低某字段的权重，可以用negative_boost

GET /forum/_search 
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "content": "java"
        }
      },
      "negative": {
        "match": {
          "content": "spark"
        }
      },
      "negative_boost": 0.2  
    }
  }
}

# negative的doc会乘以negative_boost值，降低权重。

3.constant_score

如果你压根儿不需要相关度评分，直接走constant_score加filter，所有的doc分数都是1，没有评分的概念了

GET /forum/_search 
{
  "query": {
    "bool": {
      "should": [
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "java"
              }
            }
          }
        },
        {
          "constant_score": {
            "query": {
              "match": {
                "title": "spark"
              }
            }
          }
        }
      ]
    }
  }
}
```



## 28 function score

我们可以做到自定义一个function_score函数，自己将某个field的值，跟es内置算出来的分数进行运算，然后由自己指定的field来进行分数的增强。

```text
POST /forum/_bulk
{ "update": { "_id": "1"} }
{ "doc" : {"follower_num" : 5} }
{ "update": { "_id": "2"} }
{ "doc" : {"follower_num" : 10} }
{ "update": { "_id": "3"} }
{ "doc" : {"follower_num" : 25} }
{ "update": { "_id": "4"} }
{ "doc" : {"follower_num" : 3} }
{ "update": { "_id": "5"} }
{ "doc" : {"follower_num" : 60} }

将对帖子搜索得到的分数，跟follower_num进行运算，由follower_num在一定程度上增强帖子的分数
看帖子的人越多，那么帖子的分数就越高

GET /forum/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "java spark",
          "fields": ["title", "content"]
        }
      },
      "field_value_factor": {
        "field": "follower_num",
        "modifier": "log1p",
        "factor": 0.5
      },
      "boost_mode": "sum",
      "max_boost": 2
    }
  }
}

如果只有field，那么会将每个doc的分数都乘以follower_num，如果有的doc follower是0，那么分数就会变为0，效果很不好。因此一般会加个log1p函数，公式会变为，new_score = old_score * log(1 + number_of_votes)，这样出来的分数会比较合理
再加个factor，可以进一步影响分数，new_score = old_score * log(1 + factor * number_of_votes)
boost_mode，可以决定分数与指定字段的值如何计算，multiply，sum，min，max，replace
max_boost，限制计算出来的分数不要超过max_boost指定的值

```



## 33 聚合分析

bucket和metric

bucket:按照某字段进行划分，那个字段的值相同的那些数据。

metric： 读一个数组分组执行的统计

准备数据：

```text
确定索引 mapping keyword默认开启聚合(doc_values:true)
PUT /tvs
{
	"mappings": {
		"_doc": {
			"properties": {
				"price": {
					"type": "long"
				},
				"color": {
					"type": "keyword"
				},
				"brand": {
					"type": "keyword"
				},
				"sold_date": {
					"type": "date"
				}
			}
		}
	}
}

POST /tvs/_doc/_bulk
{ "index": {}}
{ "price" : 1000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-10-28" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-11-05" }
{ "index": {}}
{ "price" : 3000, "color" : "绿色", "brand" : "小米", "sold_date" : "2016-05-18" }
{ "index": {}}
{ "price" : 1500, "color" : "蓝色", "brand" : "TCL", "sold_date" : "2016-07-02" }
{ "index": {}}
{ "price" : 1200, "color" : "绿色", "brand" : "TCL", "sold_date" : "2016-08-19" }
{ "index": {}}
{ "price" : 2000, "color" : "红色", "brand" : "长虹", "sold_date" : "2016-11-05" }
{ "index": {}}
{ "price" : 8000, "color" : "红色", "brand" : "三星", "sold_date" : "2017-01-01" }
{ "index": {}}
{ "price" : 2500, "color" : "蓝色", "brand" : "小米", "sold_date" : "2017-02-12" }
```

1.根据bucket再聚合

等价于`select avg(price),color from tvs group by color`

```text
GET /tvs/_search
{
  "size":0,
  "aggs": {
    "color_count": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "price_avg": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}

# result
"aggregations": {
    "color_count": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "price_avg": {
            "value": 3250
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "price_avg": {
            "value": 2100
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "price_avg": {
            "value": 2000
          }
        }
      ]
    }
  }
```



2.bucket嵌套实现颜色+品牌的多层下钻分析

从颜色到品牌进行下钻分析，每种颜色的平均价格，以及找到每种颜色每个品牌的平均价格。

```text
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "color_avg_price": {
          "avg": {
            "field": "price"
          }
        },
        "group_by_brand": {
          "terms": {
            "field": "brand"
          },
          "aggs": {
            "brand_avg_price": {
              "avg": {
                "field": "price"
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
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "color_avg_price": {
            "value": 3250
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "长虹",
                "doc_count": 3,
                "brand_avg_price": {
                  "value": 1666.6666666666667
                }
              },
              {
                "key": "三星",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 8000
                }
              }
            ]
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "color_avg_price": {
            "value": 2100
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "TCL",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 1200
                }
              },
              {
                "key": "小米",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 3000
                }
              }
            ]
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "color_avg_price": {
            "value": 2000
          },
          "group_by_brand": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
              {
                "key": "TCL",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 1500
                }
              },
              {
                "key": "小米",
                "doc_count": 1,
                "brand_avg_price": {
                  "value": 2500
                }
              }
            ]
          }
        }
      ]
    }
  }
```



3.统计每种颜色下单价最高、最低、平均价格

```text
GET /tvs/_search 
{
  "size": 0,
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "avg_price":{"avg": {
          "field": "price"
        }},
        "min_price":{"min": {
          "field": "price"
        }},
        "max_price":{"max": {
          "field": "price"
        }}
      }
    }
  }
}

# result
"aggregations": {
    "group_by_color": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "红色",
          "doc_count": 4,
          "max_price": {
            "value": 8000
          },
          "min_price": {
            "value": 1000
          },
          "avg_price": {
            "value": 3250
          }
        },
        {
          "key": "绿色",
          "doc_count": 2,
          "max_price": {
            "value": 3000
          },
          "min_price": {
            "value": 1200
          },
          "avg_price": {
            "value": 2100
          }
        },
        {
          "key": "蓝色",
          "doc_count": 2,
          "max_price": {
            "value": 2500
          },
          "min_price": {
            "value": 1500
          },
          "avg_price": {
            "value": 2000
          }
        }
      ]
    }
  }
```



## 47 易并行聚合算法、三角选择原则、近似聚合算法

 ```text
1.并行聚合算法：
1）易聚合分析的算法：max
2）不能聚合分析的算法：count(distinct)

2.三角选择原则：
精准+实时+大数据 --> 选择2个
1）精准+实时 没有大数据，数据量很小，随便怎么跑都可以

2）精准+大数据 hadoop 批处理，非实时

3）大数据+实时 es不精确，近似估计，可能有百分之几的错误率

3.近似聚合算法
 ```



## 52 聚合分析的原理

聚合分析时，是倒排索引+正排索引结合起来用的。

```text
GET /test_index/_search 
{
	"query": {
		"match": {
			"search_field": "test"
		}
	},
	"aggs": {
		"group_by_agg_field": {
			"terms": {
				"field": "agg_field"
			}
		}
	}
}

纯用倒排索引
hello doc1,doc2
world doc1,doc2
test1 doc1
test2 doc2
但是如果想获取age_field这个字段的值是多少，如果用倒排索引的话，需要遍历整个倒排索引，性能很差。

此时如果有正排索引
doc1 hello world test1
doc2 hello world test2
可以直接建立doc对filed值的索引文件，搜索很快。
```

```text
doc_values:文档对应的分词形成的索引文件
对部分词的所有field可以执行聚合操作。

fileddata:对分词字段的term分词进行聚合，可以打开开关，然后将正排索引数据加载到内存中，才可以对分词的field执行聚合操作，而且会消耗很大的内存。
```

 

## 53 doc_values机制内核级原理深入探秘

PUT/POST的时候，就会生成doc value数据，也就是正排索引。



**核心原理与倒排索引类似**

正排索引，也会写入磁盘文件中，然后在os cache先进行缓存，以提升访问doc value正排索引的性能。

如果os cache内存大小不足够放得下整个正排索引，会存放到磁盘文件中。



es大量基于os cache来进行缓存和提升性能的，不建议用jvm内存来进行缓存，那样会导致一定的gc开销和oom问题。因此，给jvm更少的内存，给os cache更大的内存。



doc value会进行column压缩。

**column压缩**

```text
1)所有值相同，直接保留单值
2)少于256个值，使用table encoding模式：一种压缩方式
3)大于256个值，看有没有最大公约数，有就初一最大公约数，然后保留这个最大公约数。
4）如果没有最大公约数，采取offset结合压缩的方式。 
```



## 54 string field聚合实验以及fielddata原理初探

```text
1.如果要对分词的field执行聚合操作，必须将fielddata设置为true。

2.如果对部分词的field执行聚合操作，直接就可以执行，不需要设置fielddata=true
fielddata必须在内存，因为需要执行复杂的算法和操作，如果基于磁盘和os cache那么性能会很差。
```



## 55 fielddata核心原理

fielddata加载到内存的过程是lazy加载的，对一个analyzed field执行聚合时，才会加载，而且是field-level加载的。

不是index-time创建，是query-time创建。

`indices.fielddata.cache.size: 20%` 

超出限制，清除内存已有fielddata数据。

 `circuit breaker`

如果一次query load的fielddata超过内存，就会oom -->内存溢出。



## 64 基于全局锁实现悲观锁并发控制

```text
1.线程1加锁
PUT /fs/lock/global/_create
{}

2.线程2加锁
PUT /fs/lock/global/_create
{}
会报错：
{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[lock][global]: version conflict, document already exists (current version [1])",
        "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
        "shard": "2",
        "index": "fs"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[lock][global]: version conflict, document already exists (current version [1])",
    "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
    "shard": "2",
    "index": "fs"
  },
  "status": 409
}
```

**全局锁的优点和缺点**

* 优点：操作非常简单，非常容易使用，成本低
* 缺点：你直接就把整个index给上锁了，这个时候对index中所有的doc的操作，都会被block住，导致整个系统的并发能力很低

上锁解锁的操作不是频繁，然后每次上锁之后，执行的操作的耗时不会太长，用这种方式，方便。





## 65 基于document锁实现悲观锁并发控制

全局锁，一次性就锁整个index，对这个index的所有增删改操作都会被block住，如果上锁不频繁，还可以，比较简单

细粒度的一个锁，document锁，顾名思义，每次就锁你要操作的，你要执行增删改的那些doc，doc锁了，其他线程就不能对这些doc执行增删改操作了
但是你只是锁了部分doc，其他线程对其他的doc还是可以上锁和执行增删改操作的。

document锁是用脚本进行上锁。

```text
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 123 },
  "script": "if ( ctx._source.process_id != process_id ) { assert false }; ctx.op = 'noop';"
  "params": {
    "process_id": 123
  }
}

# 解释
process_id很重要，会在lock中，设置对对应的doc加锁的进程的id，这样其他进程过来的时候，才知道，这条数据已经被别人给锁了。
assert false，不是当前进程加锁的话，则抛出异常
ctx.op='noop'，不做任何修改
```



完整的上锁过程：

```text
1.
scripts/judge-lock.groovy: if ( ctx._source.process_id != process_id ) { assert false }; ctx.op = 'noop';

POST /fs/lock/1/_update
{
  "upsert": { "process_id": 123 },
  "script": {
    "lang": "groovy",
    "file": "judge-lock", 
    "params": {
      "process_id": 123
    }
  }
}  # 对_id为1的document加锁

2.尝试改变process_id，失败
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 234 },
  "script": {
    "lang": "groovy",
    "file": "judge-lock", 
    "params": {
      "process_id": 234
    }
  }
}

# result
{
  "error": {
    "root_cause": [
      {
        "type": "remote_transport_exception",
        "reason": "[4onsTYV][127.0.0.1:9300][indices:data/write/update[s]]"
      }
    ],
    "type": "illegal_argument_exception",
    "reason": "failed to execute script",
    "caused_by": {
      "type": "script_exception",
      "reason": "error evaluating judge-lock",
      "caused_by": {
        "type": "power_assertion_error",
        "reason": "assert false\n"
      },
      "script_stack": [],
      "script": "",
      "lang": "groovy"
    }
  },
  "status": 400
}

3.删除锁
PUT /fs/lock/_bulk
{ "delete": { "_id": 1}}

4.对_id为1的document重新加锁
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 234 },
  "script": {
    "lang": "groovy",
    "file": "judge-lock", 
    "params": {
      "process_id": 234
    }
  }
}
# process_id=234上锁成功
```



## 66 基于共享锁和排它锁实现悲观锁并发控制

* 共享锁：这份数据是共享的，然后多个线程过来，都可以获取同一个数据的共享锁，然后对这个数据执行读操作
* 排他锁：排他操作，只能有一个线程获取排他锁，然后执行增删改操作。

模拟共享锁、排他锁操作：

```text
1、有人在读数据，其他人也能过来读数据

judge-lock-2.groovy: if (ctx._source.lock_type == 'exclusive') { assert false }; ctx._source.lock_count++

POST /fs/lock/1/_update 
{
  "upsert": { 
    "lock_type":  "shared",
    "lock_count": 1
  },
  "script": {
  	"lang": "groovy",
  	"file": "judge-lock-2"
  }
}

POST /fs/lock/1/_update 
{
  "upsert": { 
    "lock_type":  "shared",
    "lock_count": 1
  },
  "script": {
  	"lang": "groovy",
  	"file": "judge-lock-2"
  }
}

GET /fs/lock/1

{
  "_index": "fs",
  "_type": "lock",
  "_id": "1",
  "_version": 3,
  "found": true,
  "_source": {
    "lock_type": "shared",
    "lock_count": 3  # 可重入锁
  }
}

就给大家模拟了，有人上了共享锁，你还是要上共享锁，直接上就行了，没问题，只是lock_count加1

2、已经有人上了共享锁，然后有人要上排他锁

PUT /fs/lock/1/_create
{ "lock_type": "exclusive" }

排他锁用的不是upsert语法，create语法，要求lock必须不能存在，直接自己是第一个上锁的人，上的是排他锁

{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[lock][1]: version conflict, document already exists (current version [3])",
        "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
        "shard": "3",
        "index": "fs"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[lock][1]: version conflict, document already exists (current version [3])",
    "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
    "shard": "3",
    "index": "fs"
  },
  "status": 409
}

如果已经有人上了共享锁，明显/fs/lock/1是存在的，create语法去上排他锁，肯定会报错

3、对共享锁进行解锁

POST /fs/lock/1/_update
{
  "script": {
  	"lang": "groovy",
  	"file": "unlock-shared"
  }
}

连续解锁3次，此时共享锁就彻底没了

每次解锁一个共享锁，就对lock_count先减1，如果减了1之后，是0，那么说明所有的共享锁都解锁完了，此时就就将/fs/lock/1删除，就彻底解锁所有的共享锁

3、上排他锁，再上排他锁

PUT /fs/lock/1/_create
{ "lock_type": "exclusive" }

其他线程

PUT /fs/lock/1/_create
{ "lock_type": "exclusive" }

{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[lock][1]: version conflict, document already exists (current version [7])",
        "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
        "shard": "3",
        "index": "fs"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[lock][1]: version conflict, document already exists (current version [7])",
    "index_uuid": "IYbj0OLGQHmMUpLfbhD4Hw",
    "shard": "3",
    "index": "fs"
  },
  "status": 409
}

4、上排他锁，上共享锁

POST /fs/lock/1/_update 
{
  "upsert": { 
    "lock_type":  "shared",
    "lock_count": 1
  },
  "script": {
  	"lang": "groovy",
  	"file": "judge-lock-2"
  }
}

{
  "error": {
    "root_cause": [
      {
        "type": "remote_transport_exception",
        "reason": "[4onsTYV][127.0.0.1:9300][indices:data/write/update[s]]"
      }
    ],
    "type": "illegal_argument_exception",
    "reason": "failed to execute script",
    "caused_by": {
      "type": "script_exception",
      "reason": "error evaluating judge-lock-2",
      "caused_by": {
        "type": "power_assertion_error",
        "reason": "assert false\n"
      },
      "script_stack": [],
      "script": "",
      "lang": "groovy"
    }
  },
  "status": 400
}

5、解锁排他锁

DELETE /fs/lock/1
```





## elasticsearch 优化

1.避免超大的document

超大的document 会消耗大量内存、cpu，可能是document的几倍。

我采取的策略：

```text
1.避免对message分词索引
2.统计es内部单条document的大小，做成报表，每天输出到后端开发群，催促整改。 
```

生产es出现类似的bug：

```text
算法爬取百度百科的相关介绍保存到es中，document太大，没做过滤筛选，导致es负载过高。
```



2.避免稀疏的数据

索引尽量用相同的field名称。



**写入部分**

1.用bulk写入

我采取的策略

```text
logstash写入es的batch调大(压测调整)，一般bulk越大，压力越大。
```



2.增加refresh间隔



3.禁止swapping交换内存

`bootstrap.memory_lock=true`

官网介绍：

系统swapping的时候ES节点的性能会非常差，也会影响节点的稳定性。所以要不惜一切代价来避免swapping。swapping会导致Java GC的周期延迟从毫秒级恶化到分钟，更严重的是会引起节点响应延迟甚至脱离集群。



4.给filesystem cache更多的内存(这个东西说实话我没想到)



5.使用自动生成的id

如果要手动给document设置一个id，那么es需要每次都去确认这个id是否存在，这个过程比较耗时。



6.index buffer

如果要进行非常重的高并发写入操作，最好将index buffer调大一些

`indices.memory.index.buffer.size`设置大一些。一个shard 512m足够。



**搜索部分**

1.给filesystem cache更多内存。

```text
标准：机器留下的filesystem cache至少是数据总容量的一半。
比如说1T数据，给filesystem cache的内存至少要有512G.(这就是es集群性能不行的原因吧...)
```

elasticsearch中尽量只存用来搜索的数据，其他可以放mysql、hadoop、habase中。



2.预热filesystem cache

如果我们重启了es，那么filesystem cache是空壳的，就需要不断的查询才能重新让filesystem cache热起来，我们可以先说动对一些数据进行查询。

比如说，你本来一个查询，要用户点击以后才执行，才能从磁盘加载到filesystem cache里，第一次执行要10s，以后每次就几百毫秒

你完全可以，自己早上的时候，就程序执行那个查询，预热，数据就加载到filesystem cahce，程序执行的时候是10s，以后用户真的来看的时候就才几百毫秒



3.es查出来数据，在java python代码里面去做处理。



**优化磁盘空间**

1.禁用不需要的功能

```text
聚合：doc_values: false 正排索引 
搜索：index: false 倒排索引
评分：norms: false
近似匹配：index_options(freqs)
```



2.text转成keyword



3.禁用_all(6.x系列已取消)



4.使用best_compression

_source field和其他field都很耗费磁盘空间，最好是对其使用best_compression进行压缩。用elasticsearch.yml中的index.codec来设置，将其设置为best_compression即可。



5.用最小的最合适的数字类型

es支持4种数字类型，byte，short，integer，long。如果最小的类型就合适，那么就用最小的类型。



