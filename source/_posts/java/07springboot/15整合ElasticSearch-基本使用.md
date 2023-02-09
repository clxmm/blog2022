---
title: 15整合ElasticSearch-基本使用
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章的内容我们简单聊完 ElasticSearch 之后，本章内容我们来安装一下 ElasticSearch ，并且回顾和熟悉一下 ElasticSearch 的使用操作。

## 1. Docker安装ElasticSearch

---

为了便于快速搭建和演示，我们继续选择使用 Docker 来安装 ElasticSearch ，由于 ElasticSearch 的迭代版本多，且版本之间的特性不是很一致，本小册中我们选择使用 ElasticSearch 7.17.6 作为演示版本。

进入 Linux 服务器中，直接执行下面的命令，可以令 Docker 拉取 7.17.6 版本的 ElasticSearch ：

```css
docker run -d --name=es -p 9200:9200 -p 9300:9300 -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \n
-e "discovery.type=single-node" --privileged elasticsearch:7.17.6

```

其中的几个命令细节简单过一遍：

- `-e "ES_JAVA_OPTS=-Xms1024m -Xmx1024m"` ：默认情况下 ElasticSearch 消耗的内存较大，此处使用 JVM 参数限制最大使用内存
- `-e "discovery.type=single-node"` ：指示内部的 ElasticSearch 实例以单机模式运行
- `--privileged` ：赋予内部的 root 操作权限

启动完成后，我们可以使用 `curl localhost:9200` 检验 ElasticSearch 实例是否启动成功，当返回一组 json 数据时，即代表成功启动：

```json
{
  "name" : "4fa2c7d93f46",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "oPMri2hAT36Z3dJa7KO3lQ",
  "version" : {
    "number" : "7.17.6",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "f65e9d338dc1d07b642e14a27f338990148ee5b6",
    "build_date" : "2022-08-23T11:08:48.893373482Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

```

 

## 2. ElasticSearch的基本使用

---

ElasticSearch 在使用层面上还是比较简单的，如果我们结合上一章中的概念来使用，那样还是比较容易接受的。

下面我们来模拟一个非常经典的场景：电商网站里的商品，每个类目的不同商品有不同的型号，每个具体的商品型号又包含不同的属性。这个场景中包含 3 组一对多的关系：

- 类目 - 商品
- 商品 - 型号
- 型号 - 属性

#### 2.1 索引数据

ElasticSearch 本身也是一个存储数据的容器，要想发挥其强大的搜索能力，首先我们需要先把数据索引到 ElasticSearch 的索引中（注意这句话中，第一个索引是动词，第二个索引是名词）。ElasticSearch 本身提供了 RESTful 风格的接口，所以我们只需要发送一个 PUT 请求，即可将一条数据索引到 ElasticSearch 中：

```json
// PUT /drinks/carbonated/cola
{
    "name": "可口可乐",
    "type": "500ml",
    "price": 3.0
}
```

发送以上请求后，ElasticSearch 可以为我们返回如下数据：

```json
{
    "_index": "drinks",
    "_type": "carbonated",
    "_id": "cola",
    "_version": 1,
    "result": "created",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 0,
    "_primary_term": 1
}
```

注意观察上述返回的数据，可以发现，我们存放的数据中，`drinks` 对应的是索引，`carbonated` 对应的是类型，id 是 `cola` 。

下面我们再重复相似的操作来索引更多的数据：

```json
// PUT /drinks/carbonated/sprite
{
    "name": "雪碧柠檬味",
    "type": "500ml",
    "price": 3.0
}
```

```json
// PUT /drinks/carbonated/sprite2
{
    "name": "雪碧柠檬味",
    "type": "2L",
    "price": 6.5
}
```

```json
// PUT /drinks/carbonated/fanta
{
    "name": "芬达橙味",
    "type": "500ml",
    "price": 3.5
}
```

#### 2.2 获取单条数据

数据已经索引到 ElasticSearch 之后，下一步如果我们要获取数据的话，也是通过向 ElasticSearch 发送 http 请求，就可以得到我们刚才索引的数据。

比方说，我们要把刚才保存的“可口可乐”查出来，那就需要我们发送一个 GET 请求，uri 与索引数据时的 PUT 请求一致：

```css
curl http://localhost:9200/drinks/carbonated/cola
```

发送请求后，ElasticSearch 可以给我们返回一个类似于下面的 json 数据：

```json
{
    "_index": "drinks",
    "_type": "carbonated",
    "_id": "cola",
    "_version": 1,
    "_seq_no": 0,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "可口可乐",
        "type": "500ml",
        "price": 3.0
    }
}
```

由上述返回的数据我们可以知道，这条数据对应的核心 index 、type 、id 都与我们发送的请求一一对应，而且在索引数据的时候，发送的原始数据也都给我们返回了。

#### 2.3 删除与检查

跟上述的 uri 相同的情况下，如果我们发送的是一个 DELETE 请求，则意味着对应的数据会从 ElasticSearch 中删除。

比方说我们想删除掉“芬达橙味”，则可以发送 DELETE 请求：

```json
// DELETE /drinks/carbonated/fanta
{
    "_index": "drinks",
    "_type": "carbonated",
    "_id": "fanta",
    "_version": 2,
    "result": "deleted",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 4,
    "_primary_term": 1
}
```

当返回上述 json 数据时（注意 `result` 对应的值是 `deleted` ），即代表数据已经删除。

> 如果再发送一次同样的 DELETE 请求，会发现 result 为 not fount ，代表数据已经不存在了，无法删除。

怎么判断数据真的删除了呢？除了使用 GET 请求再次访问之外，我们还可以直接发送一个 HEAD 类型的请求，当 http 的 status （响应状态码）为 200 时，则意味着数据存在，而当返回 404 时，那就代表没有这条数据。

我们可以试着发一下 HEAD 请求，结果发现 status 为 404 Not Found ，这就证明数据已经被正确删除了。

```json
HEAD /drinks/carbonated/fanta

status: 404 Not Found
```

#### 2.4 更新数据

更新数据的方式非常简单，与索引数据相同，都是发 PUT 请求即可（类似于调用 `Map` 的 `put` 方法，即便里面有数据也可以调用 `put` 覆盖）。

比方说我们要把 500ml 的雪碧改为 888ml ，那就可以发送下面的请求：

```json
// PUT /drinks/carbonated/sprite
{
    "name": "雪碧柠檬味",
    "type": "888ml",
    "price": 4.0
}
```

发送成功后，ElasticSearch 可以给我们响应如下数据，注意观察此时 version 的值已经更新到 2 了：

```json
{
    "_index": "drinks",
    "_type": "carbonated",
    "_id": "sprite",
    "_version": 2,
    "result": "updated",
    "_shards": {
        "total": 2,
        "successful": 1,
        "failed": 0
    },
    "_seq_no": 5,
    "_primary_term": 1
}
```

当我们使用 GET 请求获取数据时，可以发现数据已经被更新了：

```json
{
    "_index": "drinks",
    "_type": "carbonated",
    "_id": "sprite",
    "_version": 2,
    "_seq_no": 5,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "name": "雪碧柠檬味",
        "type": "888ml",
        "price": 4
    }
}
```

## 3. ElasticSearch的轻量搜索

---

ElasticSearch 的强大之处还是体现在搜索上，下面我们来感受几种 ElasticSearch 的搜索方式。

#### 3.1 查全部

如果我们要获取一个 index 中的一个 type 中的所有数据，可以发送一个后缀为 **`_search`** 的请求。比方说我们可以把所有的碳酸饮料都查出来，查询出的结果不仅把当前 type 中的数据全部返回，还告诉我们一共有多少条，是以什么匹配模式返回的（本次查询为 eq 等匹配），还有最大匹配值的返回 max_score 。

```json
// GET /drinks/carbonated/_search
{
    "took": 243,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "cola",
                "_score": 1,
                "_source": {
                    "name": "可口可乐",
                    "type": "500ml",
                    "price": 3
                }
            }, {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite2",
                "_score": 1,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "2L",
                    "price": 6.5
                }
            }, {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite",
                "_score": 1,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "888ml",
                    "price": 4
                }
            }
        ]
    }
}
```

#### 3.2 模糊查

现在我们的数据中包含一个可口可乐和两个雪碧，那如果我们要把所有的雪碧都查出来，应该怎么操作呢？

还是发 GET 请求，这次在 `_search` 的请求后面需要加参数了，比方说查询所有的雪碧，那就需要加一个这样的参数：

```css
 GET /drinks/carbonated/_search ?q=name:雪碧
```

用 `属性值:value` 的方式，就代表了字符串的模糊匹配，我们可以发送一下试试看：

```json
{
    "took": 227,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.3260207,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite",
                "_score": 1.3260207,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "500ml",
                    "price": 3.0
                }
            },
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite2",
                "_score": 1.3260207,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "2L",
                    "price": 6.5
                }
            }
        ]
    }
}
```

可以发现，此时就只能查出两条数据了，并且 max_score 的值也降低了（本来就是模糊搜索，不可能 100% 匹配）。

#### 3.3 多个条件组合查询

接下来我们继续追加条件，上面的两个雪碧中一个价格是 4 块，一个是 6.5 块，比方说我们只想查不超过 5 块钱的雪碧，那就得继续在发送的请求中追加条件了：

```css
 GET /drinks/carbonated/_search ?q=+name:雪碧 +price:<5
```

可以发现，这个条件的格式变得有些奇怪，它通过两个 + 号分别指定了两个条件，后者的 **`price:<5`** 就指定了价格小于 5 块。

此时如果我们直接发送请求，会发现三个饮料全部被查出来了：

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.6983144,
        "hits": [
            // 三条数据 ......
        ]
    }
}
```

而且更奇怪的是，`max_score` 居然超过了 1 ，为什么会这样呢？

很简单，ElasticSearch 在理解我们上面发送的请求条件时，会认为这两个条件是 or 关系，不是 and 关系，所以就会导致只要有一个条件符合，就可以被成功检索出来。由此各位是否也能感受到一点：轻量搜索果然只是轻量，能实现的也忒有限了，两个条件组合都成问题（ElasticSearch 也不推荐我们在实际开发中使用这种手段）。

那如何完成多个条件的组合查询呢？

## 4. ElasticSearch的请求体参数查询

ElasticSearch 的强大检索功能体现在 请求体参数的查询 中，通过编写一个 DSL （Domain Specific Language 领域特定语言）就可以定义查询条件，DSL 可以表达的内容非常多，因此 ElasticSearch 推荐我们使用这种方式，完成实际的开发工作。

#### 4.1 单个条件的查询方式

我们还是从单个查询条件开始入手，查询所有的雪碧。这次我们发送请求时就不用带 url 参数了，而是用请求体的方式，传入一个 json 对象：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "match": {
            "name": "雪碧"
        }
    }
}
```

可以发现貌似查询条件变得难写了，但别忘了，正是由于查询结构体的规整，才使得我们可以操作的余地变得更大。

发送请求，可以发现数据依然可以正常查出。

```json
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1.3260207,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite",
                "_score": 1.3260207,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "500ml",
                    "price": 3.0
                }
            },
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite2",
                "_score": 1.3260207,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "2L",
                    "price": 6.5
                }
            }
        ]
    }
}
```

#### 4.2 多个条件and查询

下面我们来尝试查询不超过 5 块钱的雪碧，这个写法一下子复杂起来了，颇有 “十以内加减法突然跃升到高等数学” 的味道：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "name": "雪碧"
                    }
                }
            ],
            "filter": {
                "range": {
                    "price": {
                        "lt": 5
                    }
                }
            }
        }
    }
}
```

以上述的查询条件发送请求后得到的数据如下：

```json
{
    "took": 4,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.6983144,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite",
                "_score": 0.6983144,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "888ml",
                    "price": 4
                }
            }
        ]
    }
}
```

的确，这样发送请求后得到的数据是正确的，但这样一下子变化实在是有点让人头大（嗯，我也挺头大的）。下面我们来简单讲解一些其中会用到的表达式写法。

#### 4.3 match - 全文检索

全文检索是 ElasticSearch 的当家本领，我们先来说它。其实上面使用的单个条件的查询方式，本身就是全文检索的方式。但是请各位注意一个点，ElasticSearch 在执行全文检索时，会先将我们输入进去的检索内容进行分词，根据分词后的结果再来查询（比如输入“可乐雪碧”的时候，它会将名称带有 “可乐” 或 “雪碧” 任意一个词的数据全部取出），这也展现出了全文检索的特性：**相关度搜索+排序**，只要有任一词匹配，都会算在匹配结果中，而匹配的词越多，相关度会越高，搜索结果的排序也会越靠前（大家都用过搜索引擎，一定可以体会到吧）。

另外，除了使用 `match` 之外，还可以使用 `multi_match` 进行多字段检索，它的查询语法如下：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "multi_match": {
            "query": "雪碧",
            "fields": ["name", "type"]
        }
    }
}
```

#### 4.4 term精确检索 + range范围匹配

除了最常见的全文检索之外，还有一些比较常见的业务场景，诸如上面示例中的 净含量匹配、价格范围筛选 等，在这种情况下全文检索就不好使了，我们得使用另一种方式：精确检索和范围匹配。

> 净含量匹配：500ml 、888ml 、2L
>
> 价格范围筛选：3 元以下、3 - 5 元、5 元以上

下面我们来简单演示一下如何精确检索。精确检索需要使用 term 查询关键字，声明需要查询的属性，并传入要查询的值即可。下方的示例是查询净含量为 500ml 的碳酸饮料：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "term": {
            "type": {
                "value": "500ml"
            }
        }
    }
}
```

发送 GET 请求后，可以发现只查询到了一条可口可乐的数据，符合预期（雪碧已经被改为 888ml 了）。

```json
{
    "took": 3,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 0.6931471,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "cola",
                "_score": 0.6931471,
                "_source": {
                    "name": "可口可乐",
                    "type": "500ml",
                    "price": 3
                }
            }
        ]
    }
}
```

而范围查询的方式是使用 range 关键字，格式与 term 类似，只不过内部的 value 变成了比较符号（大于 小于 大于等于 小于等于 ……）。下方的示例中查询的是价格大于 3.5 的饮料：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "range": {
            "price": {
                "gt": 3.5
            }
        }
    }
}
```

执行查询，可以发现 3 块的可口可乐没有查询出来，只筛选出来了两个价格相对较高的雪碧，完全符合预期。

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 1,
        "hits": [
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite2",
                "_score": 1,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "2L",
                    "price": 6.5
                }
            },
            {
                "_index": "drinks",
                "_type": "carbonated",
                "_id": "sprite",
                "_score": 1,
                "_source": {
                    "name": "雪碧柠檬味",
                    "type": "888ml",
                    "price": 4
                }
            }
        ]
    }
}
```

#### 4.5 must&should - 复合查询

在 4.2 节中我们看到的那个复杂查询，其实就是利用的 bool 型复合查询来实现多个查询条件的组合查询。Bool 型复合查询的方式需要使用一些关键字来拼凑组合，可供使用的关键字如下：

- must ：类似于 and （且）
- should ：类似于 or （或）
- must_not ：类似于 ！（非）（注意该匹配规则不会参与相关度得分计算）
- filter ：与 must 相同，但 filter 的过滤结果不会参与相关度得分计算

下面我们再来回过头看 4.2 节的查询条件：

```json
// GET /drinks/carbonated/_search
{
    "query": {
        "bool": {
            "must": [
                {
                    "match": {
                        "name": "雪碧"
                    }
                }
            ],
            "filter": {
                "range": {
                    "price": {
                        "lt": 5
                    }
                }
            }
        }
    }
}
```

因为 Bool 型复合查询的结构就是以 bool 打头，内部包含 must 、should 等在内的 4 种关键字，而每种关键字的内部又可以组合多个子查询（包括 match 、term 、range 等），所以就有了上面的结构：参与相关度得分计算的条件是匹配饮料名带“雪碧”的，而不参与相关度得分计算的“且”匹配则是筛选价格小于 5 块的，这样组合下来就可以查出 5块钱以下的雪碧了。

更多、更高级的 DSL 查询语法和使用方式，各位可以参照 ElasticSearch 的官方文档来学习，小册不过多展开了。

【基本的操作和查询语言熟悉之后，下一章我们就来用 SpringBoot 整合 ElasticSearch ，并实际的操作一下 ElasticSearch 】

