# Elasticsearch 多字段打分规则使用与原理

## 1. 先记结论

ES 多字段打分的核心不是“字段 A 占 50%、字段 B 占 30%”这种固定百分比，而是：

```text
字段自身相关性分数
  -> 乘以字段 boost 权重
  -> 根据 multi_match / bool / dis_max / function_score 的组合规则合成最终 _score
```

常见使用方式：

```text
简单多字段搜索
  -> multi_match + fields^boost

需要每个字段不同规则
  -> bool.should + 每个 match/term 单独 boost

需要业务权重，比如热度、状态、时间、推荐
  -> function_score

只做过滤，不需要相关性
  -> bool.filter
```

## 2. `_score` 是什么

`_score` 表示当前文档和查询条件的相关性分数。

默认情况下，ES 会按 `_score` 从高到低排序。

它主要用于全文检索，例如：

```text
商品搜索
文章搜索
日志全文检索
备注内容搜索
```

结构化查询通常不需要 `_score`，例如：

```text
status = 1
waybillNumber = "SF123"
createTime >= "2026-08-01"
platformCode = "SF"
```

这类条件更适合放在 `filter` 里，只判断匹不匹配。

## 3. `_score` 的基础原理

现代 Elasticsearch 默认使用 BM25 作为相关性算法。

不用死背公式，先记三个核心因素：

```text
1. 词是否匹配
2. 词是否稀有
3. 当前字段长度是否更聚焦
```

### 3.1 词频

同一个词在字段中出现次数会影响分数。

但 BM25 有饱和机制，不会让一个词重复出现 100 次就简单变成 100 倍。

### 3.2 IDF

一个词在全体文档中越少见，区分能力越强，通常分数贡献越高。

例如搜索：

```text
Java ConcurrentHashMap
```

`ConcurrentHashMap` 比 `Java` 更有区分能力。

### 3.3 字段长度归一化

如果一个很短的字段命中了搜索词，通常说明这个字段更聚焦。

例如搜索：

```text
Elasticsearch
```

字段 A：

```text
Elasticsearch 入门
```

字段 B：

```text
一篇很长的文章，其中某处提到 Elasticsearch
```

字段 A 往往更相关。

## 4. Query Context 和 Filter Context

这是理解打分的第一分水岭。

### 4.1 Query Context

Query Context 会回答：

```text
是否匹配？
匹配程度有多高？
```

会计算 `_score`。

常见位置：

- `bool.must`
- `bool.should`
- `match`
- `multi_match`

### 4.2 Filter Context

Filter Context 只回答：

```text
是否匹配？
```

不计算相关性分数。

常见位置：

- `bool.filter`
- `bool.must_not`

示例：

```json
GET product/product_info/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "苹果手机"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": 1
          }
        }
      ]
    }
  }
}
```

含义：

```text
name 匹配“苹果手机”
  -> 参与打分

status = 1
  -> 只过滤，不参与打分
```

## 5. 多字段搜索的基本方式：multi_match

如果一个搜索词要同时查多个字段，最常用的是 `multi_match`。

例如搜索文章：

```text
title    标题
tags     标签
content  正文
```

新版本无 type 写法：

```json
GET article/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 写入",
      "fields": [
        "title^5",
        "tags^3",
        "content"
      ]
    }
  }
}
```

ES 5.x / 6.x 带 type 写法：

```json
GET article/article/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 写入",
      "fields": [
        "title^5",
        "tags^3",
        "content"
      ]
    }
  }
}
```

这里：

```text
title^5
  -> title 字段权重是 5

tags^3
  -> tags 字段权重是 3

content
  -> content 字段权重默认是 1
```

注意：`^5` 不是固定 50%，而是把该字段的相关性分数提高到更重要的位置。

## 6. Boost 不是百分比

假设原始 BM25 得分：

```text
title   = 2
tags    = 3
content = 8
```

配置：

```text
title^5
tags^3
content^1
```

可以粗略理解成：

```text
title   -> 2 * 5 = 10
tags    -> 3 * 3 = 9
content -> 8 * 1 = 8
```

但最终还要看 `multi_match` 的 `type` 如何组合分数。

所以不要说：

```text
title 占 50%
tags 占 30%
content 占 20%
```

更准确的说法是：

```text
title 比 content 更重要，权重倾向是 5:1
tags 比 content 更重要，权重倾向是 3:1
```

## 7. multi_match 的常见 type

`multi_match` 的 `type` 决定多个字段的分数如何组合。

最常用：

- `best_fields`
- `most_fields`
- `cross_fields`
- `phrase`
- `phrase_prefix`
- `bool_prefix`

日常后端开发重点掌握前三个。

## 8. best_fields

`best_fields` 是 `multi_match` 默认常见模式。

核心思想：

```text
多个字段里，哪个字段匹配最好，主要采用哪个字段的分数
```

示例：

```json
GET article/article/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 写入",
      "type": "best_fields",
      "fields": [
        "title^5",
        "content"
      ]
    }
  }
}
```

适合场景：

```text
标题、摘要、正文都可能包含搜索词
但命中某一个强字段就足够说明相关
```

例如：

```text
Doc A
title: Elasticsearch 写入原理
content: ...

Doc B
title: Java 后端总结
content: 长正文里提到 Elasticsearch 写入
```

因为 `title^5`，Doc A 通常应该排在前面。

### 8.1 tie_breaker

`best_fields` 底层类似 `dis_max` 查询。

默认更看重最佳字段。

如果希望其他字段命中也提供少量加分，可以设置 `tie_breaker`：

```json
GET article/article/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 写入",
      "type": "best_fields",
      "fields": [
        "title^5",
        "content"
      ],
      "tie_breaker": 0.3
    }
  }
}
```

粗略理解：

```text
最终分数
  = 最佳字段分数
  + 其他匹配字段分数 * tie_breaker
```

## 9. most_fields

`most_fields` 的核心思想：

```text
多个字段都匹配时，可以累加优势
```

示例：

```json
GET article/article/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 写入",
      "type": "most_fields",
      "fields": [
        "title^5",
        "tags^3",
        "content"
      ]
    }
  }
}
```

适合场景：

```text
同一个内容在多个字段都命中，说明文档更相关
```

例如：

```text
Doc A
title 命中
tags 命中
content 命中

Doc B
只有 content 命中
```

`most_fields` 会更倾向于奖励 Doc A。

## 10. cross_fields

`cross_fields` 的核心思想：

```text
把多个字段当成一个逻辑整体来查
```

典型场景：

```text
firstName = 张
lastName  = 三

用户搜索：张三
```

示例：

```json
GET user/user_info/_search
{
  "query": {
    "multi_match": {
      "query": "张三",
      "type": "cross_fields",
      "fields": [
        "firstName",
        "lastName"
      ],
      "operator": "and"
    }
  }
}
```

适合：

```text
多个字段共同组成一个逻辑字段
```

不太适合：

```text
title + tags + content
```

因为标题、标签、正文通常不是同一个逻辑字段。

注意：`cross_fields` 对 analyzer、字段长度、词频统计比较敏感，调试解释起来比普通模式更复杂。

## 11. 更可控的写法：bool.should

如果你希望每个字段有自己的查询规则，推荐使用 `bool.should`。

示例：

```json
GET article/article/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "title": {
              "query": "Elasticsearch 写入",
              "operator": "and",
              "boost": 5
            }
          }
        },
        {
          "match": {
            "tags": {
              "query": "Elasticsearch 写入",
              "boost": 3
            }
          }
        },
        {
          "match": {
            "content": {
              "query": "Elasticsearch 写入",
              "operator": "or",
              "boost": 1
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

含义：

```text
title
  -> 权重高，并且要求更严格

tags
  -> 权重中等

content
  -> 权重低，允许更宽松匹配

minimum_should_match = 1
  -> 至少命中一个 should 条件
```

这种写法比 `multi_match` 更啰嗦，但控制能力更强。

## 12. 精确字段也可以参与加分

并不是只有 `text` 字段才能影响分数。

只要放在 Query Context，比如 `should`，`term` 查询也可以参与加分。

示例：

```json
GET product/product_info/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "苹果手机"
          }
        }
      ],
      "should": [
        {
          "term": {
            "brand": {
              "value": "Apple",
              "boost": 3
            }
          }
        },
        {
          "term": {
            "isRecommended": {
              "value": true,
              "boost": 2
            }
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": 1
          }
        }
      ]
    }
  }
}
```

含义：

```text
name 匹配苹果手机
  -> 必须匹配并参与基础打分

brand = Apple
  -> 命中则加分

isRecommended = true
  -> 命中则加分

status = 1
  -> 只过滤，不加分
```

## 13. function_score

当业务排名不仅依赖文本相关性，还要考虑业务因素时，可以用 `function_score`。

常见业务因素：

- 是否推荐
- 商品销量
- 点击量
- 发布时间
- 权重等级
- 客户等级
- 是否命中特定业务状态

示例：

```json
GET product/product_info/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query": "苹果手机",
          "fields": [
            "name^5",
            "description"
          ]
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "isRecommended": true
            }
          },
          "weight": 2
        },
        {
          "field_value_factor": {
            "field": "salesCount",
            "factor": 0.01,
            "modifier": "log1p",
            "missing": 0
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "sum"
    }
  }
}
```

可以理解为：

```text
文本相关性分数
  + 推荐权重
  + 销量权重
  = 最终排名分数
```

### 13.1 score_mode

`score_mode` 控制多个 function 的分数怎么合并。

常见值：

| score_mode | 含义 |
| --- | --- |
| `multiply` | 相乘，默认 |
| `sum` | 相加 |
| `avg` | 平均 |
| `max` | 取最大 |
| `min` | 取最小 |
| `first` | 取第一个匹配函数 |

### 13.2 boost_mode

`boost_mode` 控制 function 分数和原始 query `_score` 怎么合并。

常见值：

| boost_mode | 含义 |
| --- | --- |
| `multiply` | 相乘，默认 |
| `sum` | 相加 |
| `avg` | 平均 |
| `max` | 取最大 |
| `min` | 取最小 |
| `replace` | 用 function 分数替换原始 query 分数 |

如果希望保留文本相关性，并叠加业务权重，常用：

```json
{
  "score_mode": "sum",
  "boost_mode": "sum"
}
```

如果希望完全由业务分决定排序，可以使用：

```json
{
  "boost_mode": "replace"
}
```

但这会忽略原始文本相关性，要谨慎。

## 14. 多字段分词器和打分

每个字段可以有自己的 analyzer。

例如：

```json
PUT article
{
  "mappings": {
    "article": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart"
        },
        "content": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart"
        },
        "tags": {
          "type": "text",
          "analyzer": "ik_smart"
        },
        "waybillNumber": {
          "type": "keyword"
        }
      }
    }
  }
}
```

注意：

```text
字段怎么分词
  -> 决定倒排索引里有哪些 term

查询怎么分词
  -> 决定拿哪些 term 去查

term 的分布
  -> 影响 BM25 的 IDF、TFNorm 和最终 _score
```

所以调 boost 前，先确认分词是否合理。

排查命令：

```json
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "深圳顺丰速运有限公司"
}
```

## 15. 多字段打分调优流程

建议按这条线调：

```text
1. 明确字段类型
   keyword / text / date / number

2. 明确字段用途
   精确过滤 / 全文搜索 / 排序 / 聚合 / 业务加权

3. 检查 analyzer
   用 _analyze 看真实 token

4. 先写基础查询
   multi_match 或 bool.should

5. 配置初始 boost
   标题 > 标签 > 正文 > 原始报文

6. 用 explain 看分数来源
   观察每个字段贡献

7. 根据真实搜索样本调权重
   不要只凭感觉调

8. 加入业务权重
   function_score

9. 做性能验证
   避免复杂脚本、过多字段、过大 should
```

## 16. 常见业务模板

### 16.1 商品搜索

```json
GET product/product_info/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "苹果手机",
            "type": "best_fields",
            "fields": [
              "name^5",
              "brandName^3",
              "description"
            ],
            "tie_breaker": 0.2
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": 1
          }
        }
      ],
      "should": [
        {
          "term": {
            "isRecommended": {
              "value": true,
              "boost": 2
            }
          }
        }
      ]
    }
  }
}
```

### 16.2 文章搜索

```json
GET article/article/_search
{
  "query": {
    "multi_match": {
      "query": "ES 多字段打分",
      "type": "most_fields",
      "fields": [
        "title^5",
        "summary^3",
        "tags^3",
        "content"
      ]
    }
  }
}
```

### 16.3 订单 / 运单后台查询

后台结构化查询通常不需要相关性排序。

```json
GET waybill/waybill_info/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "waybillNumber": "SF123456"
          }
        },
        {
          "term": {
            "status": 1
          }
        },
        {
          "range": {
            "createTime": {
              "gte": "2026-08-01 00:00:00"
            }
          }
        }
      ]
    }
  },
  "sort": [
    {
      "createTime": {
        "order": "desc"
      }
    }
  ]
}
```

这种场景重点是：

```text
keyword + term
date + range
bool.filter
明确 sort
```

不是 `_score`。

### 16.4 requestData 低权重搜索

如果 `requestData` 确实要参与搜索，但它只是兜底字段，建议降低权重。

```json
GET ka_receipt_subscribe_push_202405/ka_receipt_subscribe_push/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "waybillNumber": {
              "query": "顺丰签收",
              "boost": 10
            }
          }
        },
        {
          "match": {
            "customerName": {
              "query": "顺丰签收",
              "boost": 5
            }
          }
        },
        {
          "match": {
            "remark": {
              "query": "顺丰签收",
              "boost": 3
            }
          }
        },
        {
          "match": {
            "requestData": {
              "query": "顺丰签收",
              "boost": 0.3
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

如果 `requestData` 只是保存原始请求报文，不用于搜索，更推荐：

```json
{
  "requestData": {
    "type": "text",
    "index": false
  }
}
```

然后把真正要搜的字段单独拆出来。

## 17. explain 调试打分

单文档 explain 用于分析某个文档为什么匹配，以及分数怎么来的。

新版本无 type 写法：

```json
GET product/_explain/10001
{
  "query": {
    "match": {
      "name": "苹果手机"
    }
  }
}
```

ES 5.x / 6.x 带 type 写法：

```json
GET product/product_info/10001/_explain
{
  "query": {
    "match": {
      "name": "苹果手机"
    }
  }
}
```

注意：

- `_explain` 是单文档解释，通常要求明确具体 index。
- 如果 alias 指向多个 index，不能直接对这个 alias 做单文档 `_explain`。
- 可以先通过 `_search` 找到 `_index`，再对具体 index 做 `_explain`。

如果想直接看搜索结果中每条文档的解释：

```json
GET product/product_info/_search
{
  "explain": true,
  "query": {
    "match": {
      "name": "苹果手机"
    }
  }
}
```

注意：`explain=true` 输出很大，只适合调试，不要在线上常规查询中长期打开。

## 18. 你的 explain 结果怎么读

你前面看到过类似结果：

```text
requestData:和
score = 8.00704
idf = 4.797442
tfNorm = 1.6690228
docFreq = 2
docCount = 302
termFreq = 1
avgFieldLength = 1488.7053
fieldLength = 30
```

可以理解为：

```text
最终分数
  = idf * tfNorm
  = 4.797442 * 1.6690228
  ≈ 8.00704
```

高分原因：

- `和` 在当前统计范围里很少见，302 个文档只有 2 个包含，所以 IDF 高。
- 当前文档 `requestData` 字段很短，长度 30，远小于平均长度 1488.7，所以长度归一化加分明显。

这个案例最值得关注的不是 `8.00704` 本身，而是：

```text
为什么 requestData 里的“和”会进入倒排索引并参与打分？
```

如果 `requestData` 是原始 JSON 报文，通常要重新评估是否应该全文索引。

## 19. 常见坑

### 19.1 把 boost 当百分比

错误理解：

```text
title^5 = title 占 50%
```

正确理解：

```text
title 字段的相关性更重要，权重倾向提高
```

### 19.2 用 filter 做加分

`filter` 不参与打分。

如果想让某个条件命中后排名更靠前，要放在 `should`，或者使用 `function_score`。

### 19.3 对 text 字段使用 term 查询

如果字段写入时分成了：

```text
苹果
手机
```

你用：

```json
{
  "term": {
    "name": "苹果手机"
  }
}
```

可能搜不到，因为倒排索引里不一定有完整 term `苹果手机`。

### 19.4 原始报文字段参与全文打分

`requestData`、`responseData` 这类字段如果全文索引，可能产生大量无业务价值 term，导致：

- 索引膨胀
- 查询噪声
- score 异常
- merge 压力增加

### 19.5 多字段过多

`multi_match` 搜索过多字段，可能产生大量查询子句。

字段数乘以分词后的 term 数会增加查询复杂度。

### 19.6 忽略 analyzer 差异

字段 A 和字段 B 使用不同 analyzer 时，term 统计和匹配行为可能完全不同。

调 boost 前要先用 `_analyze` 看真实分词。

### 19.7 排序覆盖相关性

如果指定：

```json
{
  "sort": [
    {
      "createTime": "desc"
    }
  ]
}
```

最终排序主要按时间，不再默认按 `_score`。

如果希望相关性优先：

```json
{
  "sort": [
    {
      "_score": "desc"
    },
    {
      "createTime": "desc"
    }
  ]
}
```

## 20. 面试常问

### 20.1 多字段搜索怎么控制字段权重？

可以使用 `multi_match` 的 `fields` 配置，例如：

```json
{
  "fields": [
    "title^5",
    "tags^3",
    "content"
  ]
}
```

也可以使用 `bool.should`，给每个字段单独写 `match` 并配置 `boost`。

### 20.2 boost 是百分比吗？

不是。

Boost 是相关性权重倍数或倾向，不是固定占比。

最终分数还会受 BM25、字段长度、词频、IDF、multi_match type、业务函数等影响。

### 20.3 best_fields 和 most_fields 区别？

`best_fields` 更看重单个最佳匹配字段。

`most_fields` 会组合多个字段的匹配分数，更奖励多个字段同时命中。

### 20.4 should 一定会扩大结果集吗？

不一定。

如果 `bool` 中已经有 `must` 或 `filter`，`should` 默认更多是加分项。

如果只有 `should`，通常至少要匹配一个 `should`。

可以用 `minimum_should_match` 明确控制。

### 20.5 filter 为什么更适合后台查询？

后台查询很多是结构化条件，只关心是否匹配，不关心相关性。

`filter` 不计算 `_score`，更符合业务语义，也有利于缓存。

### 20.6 function_score 解决什么问题？

解决文本相关性之外的业务排序问题。

例如：

```text
文本相关性
+ 推荐权重
+ 销量
+ 发布时间
+ 客户等级
```

## 21. 一句话总结

ES 多字段打分可以这样记：

```text
match / multi_match 负责文本相关性
boost 负责字段重要程度
bool.should 负责命中加分
bool.filter 负责无打分过滤
function_score 负责叠加业务排序
explain 负责拆解每个分数来源
```

真正调优时不要追求固定百分比，而是基于真实搜索样本、真实 analyzer、真实 explain 结果，逐步调整字段权重和业务权重。

## 22. 参考来源

### 22.1 本次输入背景

- ChatGPT 对话：`ES类型和常用命令`
  - 用作本笔记的问题背景，重点参考其中关于 `_score`、分词器、多字段权重、`_explain` 的讨论。

### 22.2 Elastic 官方文档

- Multi-match query
  - https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-multi-match-query
- Boolean query
  - https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query
- Function score query
  - https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-function-score-query
- Similarity settings
  - https://www.elastic.co/docs/reference/elasticsearch/index-settings/similarity
- Explain API
  - https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-explain
- Ranking and reranking
  - https://www.elastic.co/docs/solutions/search/ranking

