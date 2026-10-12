# Elasticsearch 评分、二次评分与排序 Java 总结

## 目录

- [1. 核心结论](#1-核心结论)
- [2. 搜索评分完整链路](#2-搜索评分完整链路)
- [3. 第一阶段相关性评分](#3-第一阶段相关性评分)
- [4. Query Context 与 Filter Context](#4-query-context-与-filter-context)
- [5. 多字段搜索与字段权重](#5-多字段搜索与字段权重)
- [6. Bool Should 加分](#6-bool-should-加分)
- [7. Function Score 业务评分](#7-function-score-业务评分)
- [8. Rescore 二次评分原理](#8-rescore-二次评分原理)
- [9. Rescore 类型与版本边界](#9-rescore-类型与版本边界)
- [10. Query Rescorer 的查询方式](#10-query-rescorer-的查询方式)
- [11. Rescore 分数合并方式](#11-rescore-分数合并方式)
- [12. 多阶段 Rescore](#12-多阶段-rescore)
- [13. ES 5.x Java 完整示例](#13-es-5x-java-完整示例)
- [14. 公司 ES Client 接入方式](#14-公司-es-client-接入方式)
- [15. 排序 Sort](#15-排序-sort)
- [16. Post Filter、Min Score 与 Track Scores](#16-post-filtermin-score-与-track-scores)
- [17. Mapping 与分词要求](#17-mapping-与分词要求)
- [18. Explain 与评分排查](#18-explain-与评分排查)
- [19. 性能与分页注意事项](#19-性能与分页注意事项)
- [20. Java 应用侧精排](#20-java-应用侧精排)
- [21. 方案选择](#21-方案选择)
- [22. 常见误区](#22-常见误区)
- [23. 面试总结](#23-面试总结)
- [24. 参考资料](#24-参考资料)

## 1. 核心结论

Elasticsearch 的搜索排名可以概括为：

```text
Query 召回
    ↓
BM25 计算第一阶段 _score
    ↓
boost / should / function_score 调整相关性
    ↓
每个分片取 Top window_size
    ↓
Rescore 二次评分
    ↓
按最终 _score DESC 返回 Top N
```

主要能力的职责如下：

| 能力 | 解决的问题 |
| --- | --- |
| BM25 / `_score` | 文档与查询词有多相关 |
| `boost` | 哪个字段或条件更重要 |
| `bool.should` | 命中额外条件后提高分数 |
| `bool.filter` | 只筛选，不参与相关性评分 |
| `function_score` | 将销量、时间、等级等业务因素加入评分 |
| `rescore` | 只对第一阶段 Top N 候选进行昂贵精排 |
| `sort` | 决定结果最终展示顺序 |
| `_explain` | 分析某个文档为什么得到当前分数 |
| `min_score` | 排除相关性过低的文档 |
| `track_scores` | 字段排序时仍计算并返回 `_score` |

推荐记忆：

```text
Query = 召回与粗排
Rescore = 候选集精排
Sort = 最终结果顺序
```

## 2. 搜索评分完整链路

一次相关性搜索大致经历以下过程：

```text
用户输入关键词
    ↓
查询词经过 search_analyzer 分词
    ↓
match / multi_match / bool 查询倒排索引
    ↓
BM25 计算基础 _score
    ↓
boost、should、function_score 调整分数
    ↓
每个分片保留局部 Top K
    ↓
可选：Rescore 对局部候选重新评分
    ↓
协调节点归并各分片结果
    ↓
返回最终 Top N
```

Rescore 不能召回第一阶段没有进入候选窗口的文档。因此，第一阶段查询必须保证足够的召回率。

## 3. 第一阶段相关性评分

### 3.1 `_score`

使用 `match`、`multi_match`、`query_string` 等全文查询时，Elasticsearch 会给每个匹配文档计算 `_score`。

没有显式排序时，相关性搜索默认近似等价于：

```text
_score DESC
```

### 3.2 BM25

ES 5.x 默认使用 BM25。可以粗略理解为：

```text
_score
≈ TF 词频贡献
× IDF 词项稀有程度
× 字段长度修正
× boost
```

主要影响因素：

| 因素 | 含义 |
| --- | --- |
| TF | 查询词在当前文档出现得更多时通常分数更高，但增长会逐渐饱和 |
| IDF | 查询词越稀有，区分能力越强，权重通常越高 |
| 字段长度 | 同样命中一次，短字段通常比超长字段更聚焦 |
| Boost | 人为调整字段、查询子句或业务条件的重要性 |

例如搜索 `Java`，一个字段中出现 100 次并不会简单获得另一个字段的 100 倍分数，这是 BM25 词频饱和的体现。

### 3.3 分数不可跨查询直接比较

`_score` 是当前查询、当前索引统计和当前分片环境下的相对值。

不要将不同关键词、不同索引、不同时间执行的 `_score` 当成统一绝对分数比较。

## 4. Query Context 与 Filter Context

### 4.1 Query Context

Query Context 同时回答：

```text
是否匹配？
匹配程度有多高？
```

常见位置：

- `bool.must`
- `bool.should`
- `match`
- `multi_match`
- `function_score`

这些查询通常会参与 `_score` 计算。

### 4.2 Filter Context

Filter Context 只回答：

```text
是否匹配？
```

常见位置：

- `bool.filter`
- `bool.must_not`

适合放入过滤上下文的条件：

- 状态
- 类型
- 租户 ID
- 分类 ID
- 是否删除
- 时间范围
- 运单号、接口编码等精确标识

示例：

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "苹果手机"
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

其中 `title` 参与相关性评分，`status` 只过滤。

## 5. 多字段搜索与字段权重

### 5.1 Multi Match

一个关键词搜索多个字段时，可以使用 `multi_match`：

```json
{
  "query": {
    "multi_match": {
      "query": "苹果手机",
      "fields": [
        "title^5",
        "brand^3",
        "description"
      ]
    }
  }
}
```

权重关系为：

```text
title       5
brand       3
description 1
```

`boost=5` 不是固定增加 5 分，也不是字段占比 50%，而是放大该查询子句产生的相关性分数。

### 5.2 Multi Match 常见类型

| 类型 | 分数组合特点 | 典型场景 |
| --- | --- | --- |
| `best_fields` | 主要采用得分最高的字段，可用 `tie_breaker` 引入其他字段少量贡献 | 标题、摘要、正文命中任一强字段即可 |
| `most_fields` | 多个字段命中可以共同贡献分数 | 多个分析字段或标题、标签、正文共同命中 |
| `cross_fields` | 将多个字段视作一个逻辑字段 | 姓名拆分、地址拆分 |
| `phrase` | 对每个字段执行短语查询，通常仍偏向最佳字段 | 多字段相同短语规则 |
| `phrase_prefix` | 短语最后一个词允许前缀 | 搜索联想 |

`best_fields` 不能简单理解为所有字段得分相加。它底层更接近 `dis_max`：主要采用最佳字段分数。

如果要求多个字段都贡献分数，优先考虑 `bool.should`。

## 6. Bool Should 加分

`bool.should` 适合每个字段使用不同查询方式或不同权重：

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "签收"
          }
        }
      ],
      "should": [
        {
          "term": {
            "important": {
              "value": 1,
              "boost": 5
            }
          }
        },
        {
          "term": {
            "source": {
              "value": "official",
              "boost": 3
            }
          }
        }
      ]
    }
  }
}
```

其语义是：

```text
content 必须匹配“签收”
important=1 命中后额外贡献分数
source=official 命中后额外贡献分数
```

如果一个 `bool` 只有 `should`，通常至少需要命中一个。为了避免语义随其他子句变化，建议显式设置：

```json
"minimum_should_match": 1
```

## 7. Function Score 业务评分

### 7.1 适用场景

当排序不仅依赖文本相关性，还需要考虑销量、热度、新鲜度、等级或推荐状态时，可以使用 `function_score`。

```text
最终相关性
= 文本 BM25
+ 销量贡献
+ 推荐权重
+ 时间衰减
```

### 7.2 主要评分函数

| 评分函数 | 用途 |
| --- | --- |
| `weight` | 满足条件时增加固定权重 |
| `field_value_factor` | 使用数值字段参与评分 |
| `gauss` | 高斯衰减，常用于时间、距离和数值新鲜度 |
| `exp` | 指数衰减 |
| `linear` | 线性衰减 |
| `random_score` | 随机打散或实验流量 |
| `script_score` | 用 Painless 自定义评分公式 |

示例：

```json
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "name": "苹果手机"
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "isVip": true
            }
          },
          "weight": 2
        },
        {
          "field_value_factor": {
            "field": "sales",
            "factor": 0.1,
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

### 7.3 Function Score 的两层合并

`score_mode` 控制多个评分函数如何合并：

| `score_mode` | 含义 |
| --- | --- |
| `multiply` | 多个函数分数相乘 |
| `sum` | 多个函数分数相加 |
| `avg` | 取平均值 |
| `max` | 取最大值 |
| `min` | 取最小值 |
| `first` | 使用第一个匹配函数 |

`boost_mode` 控制函数分数与原始查询分数如何合并：

| `boost_mode` | 含义 |
| --- | --- |
| `multiply` | 原始分数与函数分数相乘 |
| `sum` | 两部分相加 |
| `avg` | 取平均值 |
| `max` | 取最大值 |
| `min` | 取最小值 |
| `replace` | 用函数分数替换原始分数 |

不要直接执行：

```text
_score + sales
```

销量可能达到十万甚至百万，会完全淹没文本相关性。通常使用：

```text
log1p(sales)
sqrt(sales)
归一化后的 CTR
```

## 8. Rescore 二次评分原理

### 8.1 为什么需要 Rescore

短语查询、复杂脚本和机器学习模型通常比普通倒排查询昂贵。

如果对所有候选文档执行复杂评分，性能成本会非常高。Rescore 的思想是：

```text
大量文档
    ↓
快速 Query 召回与粗排
    ↓
每个分片取 Top window_size
    ↓
只对候选集执行复杂精排
```

### 8.2 Window Size

```json
"window_size": 500
```

表示每个分片参与精排的候选文档数量，不是整个集群全局只精排 500 条。

如果索引包含 5 个分片，最多可能有约：

```text
500 × 5 = 2500
```

个分片级候选参与精排与归并。

`window_size` 越大，精排召回越充分，但 CPU、内存和查询耗时也越高。

## 9. Rescore 类型与版本边界

### 9.1 ES 5.x

ES 5.x Java API 的核心实现是：

```text
Query Rescorer
QueryRescorerBuilder
```

因此，ES 5.x 的学习重点是将各种 Query DSL 放入 `rescore_query`。

### 9.2 当前新版 Elasticsearch

当前 Elasticsearch 文档列出三类 Rescorer：

| 类型 | 作用 |
| --- | --- |
| Query Rescorer | 对 Top K 执行第二个查询 |
| Script Rescorer | 脚本直接读取 `_score` 与文档字段并重新计算 |
| Learning To Rank Rescorer | 使用已部署的 LTR 模型重新排序 |

内置 LTR Rescorer 是 Elasticsearch 8.12.0 后的能力，并受订阅级别限制。

ES 5.x 不能照搬新版独立 `script` 或 `learning_to_rank` Rescorer 写法。

ES 5.x 如需脚本精排，应采用：

```text
Query Rescorer
    ↓
rescore_query
    ↓
function_score
    ↓
script_score
```

## 10. Query Rescorer 的查询方式

`rescore_query` 是完整 Query DSL，可以放入单字段查询、多字段查询、布尔查询和函数评分查询。

### 10.1 单字段短语精排

```json
"rescore_query": {
  "match_phrase": {
    "title": {
      "query": "苹果手机",
      "slop": 0,
      "boost": 5
    }
  }
}
```

适合标题、名称等强字段完整短语优先。

### 10.2 多字段短语精排

推荐使用一个 `bool.should`：

```json
"rescore_query": {
  "bool": {
    "should": [
      {
        "match_phrase": {
          "title": {
            "query": "苹果手机",
            "slop": 0,
            "boost": 5
          }
        }
      },
      {
        "match_phrase": {
          "brand": {
            "query": "苹果手机",
            "slop": 0,
            "boost": 3
          }
        }
      },
      {
        "match_phrase": {
          "description": {
            "query": "苹果手机",
            "slop": 2,
            "boost": 1
          }
        }
      }
    ],
    "minimum_should_match": 1
  }
}
```

特点：

- 一次 Rescore 同时处理多个字段。
- 每个字段可以设置不同权重。
- 每个字段可以设置不同 `slop`。
- 多个字段命中时可以共同贡献分数。
- `should` 是加分和匹配条件，不要求全部字段同时命中。

### 10.3 Multi Match Phrase

多个字段使用完全相同的短语规则时，可以简写为：

```json
"rescore_query": {
  "multi_match": {
    "query": "苹果手机",
    "fields": [
      "title^5",
      "brand^3",
      "description"
    ],
    "type": "phrase",
    "slop": 1
  }
}
```

`type=phrase` 虽然会对多个字段执行短语查询，但字段合并通常仍偏向最佳字段，不等价于所有字段得分简单累加。

需要多字段共同贡献时，使用 `bool.should + 多个 match_phrase` 更直观。

### 10.4 混合查询精排

不同字段可以采用不同查询方式：

```json
"rescore_query": {
  "bool": {
    "should": [
      {
        "match_phrase": {
          "title": {
            "query": "苹果手机",
            "boost": 5
          }
        }
      },
      {
        "term": {
          "brand.keyword": {
            "value": "Apple",
            "boost": 3
          }
        }
      },
      {
        "match_phrase_prefix": {
          "description": {
            "query": "苹果手",
            "boost": 0.5
          }
        }
      }
    ]
  }
}
```

常见可组合查询：

| 查询 | 精排用途 |
| --- | --- |
| `match_phrase` | 完整短语优先 |
| `match_phrase_prefix` | 搜索联想、最后一个词前缀 |
| `term` | 品牌、编码、状态精确命中 |
| `prefix` | 编码或名称前缀 |
| `span_near` | 精细控制词项顺序和距离 |
| `bool.should` | 多字段、多规则共同加分 |
| `constant_score` | 命中业务条件后给予固定查询分数 |

### 10.5 Constant Score 固定业务加分

普通 `boost` 会放大原始相关性。如果需要命中某个业务条件后获得相对固定的分数，可以使用 `constant_score`：

```json
"rescore_query": {
  "bool": {
    "should": [
      {
        "match_phrase": {
          "title": {
            "query": "苹果手机",
            "boost": 5
          }
        }
      },
      {
        "constant_score": {
          "filter": {
            "term": {
              "isVip": true
            }
          },
          "boost": 3
        }
      }
    ]
  }
}
```

### 10.6 Function Score 精排

`function_score` 可以直接作为 `rescore_query`：

```text
第一阶段：BM25 粗排
第二阶段：短语、销量、热度、时间等业务评分
```

适用函数包括：

- `weight`
- `field_value_factor`
- `gauss` / `exp` / `linear`
- `random_score`
- `script_score`

### 10.7 Script Score 自定义精排

ES 5.x 可以通过 `function_score` 内的 `script_score` 实现复杂公式，例如：

```text
文本质量分
+ log1p(点击量) × 0.1
+ 业务等级 × 0.5
- 超时惩罚
```

Java 主要使用：

```java
QueryBuilders.functionScoreQuery(...)
ScoreFunctionBuilders.scriptFunction(...)
```

脚本使用要求：

- 优先读取支持 `doc_values` 的字段。
- 避免在评分脚本中读取 `_source`。
- 业务参数通过 `params` 传递，不动态拼接脚本字符串。
- 使用 `log1p`、`sqrt`、归一化等方式控制数值范围。
- 将脚本限制在较小的 `window_size` 中执行。

## 11. Rescore 分数合并方式

设：

```text
Q = 第一阶段 queryScore × query_weight
R = 第二阶段 rescoreScore × rescore_query_weight
```

Query Rescorer 支持以下 `score_mode`：

| `score_mode` | 近似公式 | 适用场景 |
| --- | --- | --- |
| `total` | `Q + R` | 文本粗排分与精排分累加，默认且最常用 |
| `multiply` | `Q × R` | 用业务质量放大或削弱原始相关性 |
| `avg` | `(Q + R) / 2` | 平衡两阶段评分 |
| `max` | `max(Q, R)` | 任一阶段高分即可 |
| `min` | `min(Q, R)` | 两阶段都要表现较好 |

示例：

```json
"query_weight": 0.7,
"rescore_query_weight": 2.0,
"score_mode": "total"
```

若某文档：

```text
queryScore   = 8
rescoreScore = 10
```

则：

```text
finalScore
= 8 × 0.7 + 10 × 2.0
= 25.6
```

两层权重需要分开理解：

```text
title^5 / brand^3 / description^1
    -> 控制 Rescore 内部各字段的重要程度

query_weight / rescore_query_weight
    -> 控制第一阶段与第二阶段的重要程度
```

## 12. 多阶段 Rescore

多个字段不等于必须使用多个 Rescore。通常一个 `rescore_query + bool.should` 就足够。

只有真正存在多阶段精排时，才使用 Rescore 数组：

```json
"rescore": [
  {
    "window_size": 500,
    "query": {
      "rescore_query": {
        "match_phrase": {
          "title": "苹果手机"
        }
      },
      "query_weight": 1,
      "rescore_query_weight": 3,
      "score_mode": "total"
    }
  },
  {
    "window_size": 100,
    "query": {
      "rescore_query": {
        "function_score": {
          "query": {
            "match_all": {}
          },
          "field_value_factor": {
            "field": "sales",
            "modifier": "log1p",
            "factor": 0.1,
            "missing": 0
          }
        }
      },
      "query_weight": 1,
      "rescore_query_weight": 1,
      "score_mode": "multiply"
    }
  }
]
```

执行顺序：

```text
普通 Query
    ↓
Top 500
    ↓
第一阶段：标题短语精排
    ↓
Top 100
    ↓
第二阶段：销量业务精排
    ↓
最终结果
```

通常前一阶段窗口更大、规则较简单；后一阶段窗口更小、计算更昂贵。

## 13. ES 5.x Java 完整示例

下面以 Java 8 和旧版 Elasticsearch Builder API 为例，实现“宽松召回 + 多字段短语精排”。

### 13.1 构建粗排查询

```java
String keyword = "苹果手机";

MultiMatchQueryBuilder recallQuery = QueryBuilders.multiMatchQuery(
        keyword,
        "title",
        "brand",
        "description"
);
```

### 13.2 构建多字段短语精排

```java
BoolQueryBuilder phraseRescoreQuery = QueryBuilders.boolQuery()
        .should(QueryBuilders.matchPhraseQuery("title", keyword)
                .slop(0)
                .boost(5.0F))
        .should(QueryBuilders.matchPhraseQuery("brand", keyword)
                .slop(0)
                .boost(3.0F))
        .should(QueryBuilders.matchPhraseQuery("description", keyword)
                .slop(2)
                .boost(1.0F))
        .minimumShouldMatch(1);
```

### 13.3 构建 Query Rescorer

```java
QueryRescorerBuilder rescorerBuilder =
        new QueryRescorerBuilder(phraseRescoreQuery);

rescorerBuilder.windowSize(500);
rescorerBuilder.setQueryWeight(1.0F);
rescorerBuilder.setRescoreQueryWeight(2.0F);
rescorerBuilder.setScoreMode(QueryRescoreMode.Total);
```

### 13.4 构建 Search Request

```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
        .query(recallQuery)
        .from(0)
        .size(20)
        .addRescorer(rescorerBuilder)
        .sort(SortBuilders.scoreSort().order(SortOrder.DESC));

SearchRequest searchRequest = new SearchRequest("products")
        .source(sourceBuilder);
```

不同 Elasticsearch Java Client 版本的执行方法和包名可能不同，但 Query DSL 结构保持一致。

## 14. 公司 ES Client 接入方式

公司 `es-client` 的 `GenericQuery` / `SearchBO` 主要覆盖：

- `term`
- `match`
- `range`
- `wildcard`
- `exists`
- `terms`
- 普通分页、Scroll、Search After

当前封装没有明确暴露 Rescore 模型。复杂精排建议获取原生客户端：

```java
@Resource
private ESClient esClient;

RestHighLevelClient client = esClient.getRestHighLevelClient();
```

然后使用：

```text
SearchSourceBuilder
+ QueryRescorerBuilder
+ 原生 QueryBuilders
```

如果底层客户端版本无法表达某个 DSL，也可以使用：

```java
esClient.getRestClient()
```

直接提交完整 JSON 请求。

使用前需要确认：

```text
1. ES 服务端真实版本
2. elasticsearch-rest-high-level-client 版本
3. 公司 es-client 封装所依赖的 ES Client 版本
```

## 15. 排序 Sort

### 15.1 普通相关性排序

未指定字段排序时，全文查询通常按：

```text
_score DESC
```

### 15.2 普通查询的多字段排序

不使用 Rescore 时，可以配置：

```json
"sort": [
  {
    "_score": {
      "order": "desc"
    }
  },
  {
    "createTime": {
      "order": "desc"
    }
  },
  {
    "id": {
      "order": "asc"
    }
  }
]
```

含义：

```text
第一优先级：相关度
第二优先级：创建时间
第三优先级：ID
```

### 15.3 使用 Rescore 时的限制

使用 Rescore 时，只能按 `_score DESC` 排序。

如果显式设置其他字段排序，例如：

```json
"sort": [
  {
    "createTime": "desc"
  }
]
```

或者：

```json
"sort": [
  { "_score": "desc" },
  { "createTime": "desc" }
]
```

通常会与 Rescore 冲突并报错。

因此需要明确选择：

```text
需要精排相关性
    -> Rescore + _score DESC

用户明确选择最新、价格、销量排序
    -> 不使用 Rescore，直接字段 sort
```

## 16. Post Filter、Min Score 与 Track Scores

### 16.1 Post Filter

`post_filter` 在查询与聚合后过滤最终 hits，常用于搜索导航：

```json
{
  "query": {
    "match": {
      "title": "手机"
    }
  },
  "post_filter": {
    "term": {
      "brand": "Apple"
    }
  }
}
```

普通业务过滤优先使用 `bool.filter`。只有聚合需要看到未过滤分布、hits 又需要过滤时，才考虑 `post_filter`。

### 16.2 Min Score

```json
{
  "min_score": 5,
  "query": {
    "match": {
      "name": "苹果手机"
    }
  }
}
```

只返回最终 `_score >= 5` 的文档。

生产环境慎用固定阈值，因为 `_score` 会随查询词、分词、索引统计和分片环境变化。

### 16.3 Track Scores

字段排序时，ES 可能不再需要计算 `_score`。如果仍希望返回评分，可以设置：

```json
{
  "track_scores": true,
  "query": {
    "match": {
      "title": "苹果"
    }
  },
  "sort": [
    {
      "createTime": "desc"
    }
  ]
}
```

`track_scores=true` 只是让 ES 计算并返回 `_score`，不代表最终按 `_score` 排序。

该字段排序场景不应与 Rescore 混用。

## 17. Mapping 与分词要求

### 17.1 Match Phrase 依赖词项位置

`match_phrase` 依赖分词后的词项及其位置信息，字段通常应为 `text`：

```json
"title": {
  "type": "text",
  "analyzer": "ik_max_word",
  "search_analyzer": "ik_smart"
}
```

### 17.2 Keyword 字段

`keyword` 字段通常把整个值当作一个 Token，更适合：

- `term`
- 排序
- 聚合
- 精确业务加分

例如：

```json
"brand": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword"
    }
  }
}
```

使用方式：

```text
brand         -> 全文或短语查询
brand.keyword -> 精确 term 查询
```

### 17.3 Slop

`slop` 控制短语中词项允许的位置偏移：

```text
slop=0：要求词项顺序和位置严格相邻
slop=1/2：允许一定插词或位置移动
```

一般建议：

```text
标题：较小 slop
描述：适度放宽 slop
```

调权重前，先用 `_analyze` 检查真实分词结果：

```json
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "苹果手机"
}
```

## 18. Explain 与评分排查

### 18.1 单文档 Explain

ES 5.x 带 Type 的写法：

```json
GET products/product/10001/_explain
{
  "query": {
    "match": {
      "title": "苹果手机"
    }
  }
}
```

用于分析：

- 查询词被分成哪些 Term。
- 每个 Term 的 TF、IDF 是多少。
- 字段长度如何影响分数。
- Boost 如何参与计算。

### 18.2 Search Explain

```json
GET products/product/_search
{
  "explain": true,
  "query": {
    "match": {
      "title": "苹果手机"
    }
  }
}
```

`explain=true` 输出很大，只适合小规模调试，不要在线上常规请求中长期启用。

### 18.3 Profile

查询耗时异常时，可以使用 Profile API 分析查询阶段开销。

需要注意，Profile 本身会增加明显开销，只能用于诊断环境或受控请求。

### 18.4 建议排查顺序

```text
1. 检查 Mapping 类型
2. 用 _analyze 检查写入与查询分词
3. 去掉 Rescore 验证粗排候选
4. 单独验证 rescore_query 是否匹配
5. 打开 explain 检查字段贡献
6. 检查 query_weight、rescore_query_weight、score_mode
7. 调整 window_size 并比较延迟和排序质量
8. 使用真实标注样本做回归测试
```

## 19. 性能与分页注意事项

### 19.1 Window Size 是每个分片的窗口

总成本近似受以下因素影响：

```text
分片数
× window_size
× rescore_query 单文档计算成本
```

### 19.2 分页保持 Window Size 固定

翻页时不要随意改变 `window_size`，否则候选集和重排结果会变化，用户可能看到结果跳动。

通常应保证：

```text
window_size >= from + size
```

至少让当前页面范围内的文档都经历相同精排。

深分页不适合无限增大 `window_size`。应结合业务限制最大页数，或重新设计分页与召回方案。

### 19.3 召回查询不要过严

Rescore 只能重排第一阶段已经召回的文档。

推荐模式：

```text
粗排：match / multi_match，扩大召回
精排：match_phrase / function_score / script，提升精度
```

### 19.4 评分字段使用 Doc Values

数值、时间、等级等业务字段应适合通过 `doc_values` 读取。

避免评分脚本频繁解析 `_source`。

### 19.5 建立评测集

不要只凭肉眼调整权重。应准备：

```text
查询词
期望 Top 结果
相关等级
错误样本
```

然后对比不同 `boost`、`slop`、`window_size` 和 `score_mode` 的效果。

## 20. Java 应用侧精排

Java 应用侧精排不属于 Elasticsearch `rescore` API，但也是常见二次排序方案：

```text
ES 召回 Top 500
    ↓
Java 获取候选特征
    ↓
规则或模型计算 finalScore
    ↓
Java 重新排序
    ↓
返回 Top 20
```

优点：

- 可以调用任意规则、远程服务或机器学习模型。
- 不受 ES Query DSL 限制。
- 业务逻辑迭代灵活。

缺点：

- 需要传输更多候选数据。
- 应用内存和延迟成本更高。
- 分页、排序稳定性和总数语义需要自行处理。
- 模型服务失败时需要降级到 ES 原始排序。

如果 ES 5.x 无法满足复杂 LTR 或语义重排需求，可以考虑应用侧精排，但必须设置超时、熔断和降级策略。

## 21. 方案选择

| 业务需求 | 推荐方案 |
| --- | --- |
| 结构化后台查询 | `bool.filter + 字段 sort` |
| 普通全文搜索 | `match / multi_match + _score DESC` |
| 不同字段不同权重 | `bool.should + boost` |
| 多字段完整短语优先 | 一个 Rescore + 多个 `match_phrase` |
| 文本相关性叠加销量、等级 | `function_score` |
| 只对 Top N 计算业务分 | `rescore_query + function_score` |
| 多级由易到难精排 | 多个 Rescore，窗口逐级缩小 |
| 复杂自定义数学模型 | `script_score`，严格控制窗口和性能 |
| LTR / 语义模型 | 新版内置能力或 Java 应用侧精排 |
| 用户明确选择最新、价格最低 | 不使用 Rescore，直接字段排序 |

推荐的多字段短语搜索方案：

```text
multi_match 宽松召回
    ↓
一个 QueryRescorerBuilder
    ↓
bool.should
    ├─ title match_phrase boost=5 slop=0
    ├─ brand match_phrase boost=3 slop=0
    └─ description match_phrase boost=1 slop=2
    ↓
score_mode=total
    ↓
_score DESC
```

## 22. 常见误区

### 22.1 Boost 等于固定加分

错误：

```text
boost=5 就是固定加 5 分
```

正确：

```text
boost 会调整查询子句产生的相关性分数
```

需要相对固定业务分时，可考虑 `constant_score` 或 `function_score weight`。

### 22.2 多字段必须配置多个 Rescore

错误。

一个 `rescore_query` 可以包含完整 `bool.should`，一次精排多个字段。

多个 Rescore 用于真正的多阶段精排，而不是简单按字段拆分。

### 22.3 Should 要求所有字段命中

错误。

`should` 通常表示命中一个即可，并且多命中可以继续贡献分数。要求全部命中应使用 `must`，但会明显降低召回率。

### 22.4 Multi Match Phrase 等于多字段分数累加

错误。

`type=phrase` 更接近多字段 Phrase 查询的最佳字段组合。明确要求所有字段贡献时，优先使用 `bool.should`。

### 22.5 Rescore 可以召回遗漏文档

错误。

Rescore 只能处理进入 `window_size` 的候选文档，不能补召回。

### 22.6 Rescore 后还能按时间或销量排序

错误。

Rescore 要求最终按 `_score DESC`。用户选择字段排序时，应跳过 Rescore。

### 22.7 Track Scores 会改变字段排序顺序

错误。

`track_scores=true` 只负责计算并返回 `_score`，最终仍按显式字段排序。

### 22.8 Min Score 是稳定阈值

错误。

不同查询的 `_score` 不可简单横向比较，固定 `min_score` 需要用真实数据充分验证。

### 22.9 原始报文字段适合高权重全文搜索

`requestData`、`responseData` 等大字段通常噪声很多，容易造成：

- 索引膨胀
- 无意义 Term 增多
- 相关性异常
- 查询和合并压力增加

如果确需搜索，应拆出业务字段，或给原始报文字段设置很低权重。

## 23. 面试总结

### 23.1 一句话回答

Elasticsearch 全文检索先通过 BM25 计算 `_score`，再通过字段 Boost、`bool.should` 和 `function_score` 融入字段与业务权重；复杂精排可以使用 Rescore，仅对每个分片第一阶段 Top N 候选执行短语、函数或脚本评分，最后按 `_score` 降序返回结果。

### 23.2 多字段短语精排回答

一次 Rescore 可以同时精排多个字段。Java 中通常使用 `QueryRescorerBuilder`，将 `bool.should` 作为 `rescore_query`，在其中为 `title`、`brand`、`description` 分别配置 `matchPhraseQuery`、`boost` 和 `slop`。多个字段命中时共同贡献精排分数，再通过 `query_weight`、`rescore_query_weight` 和 `score_mode` 与第一阶段分数合并。

### 23.3 Function Score 与 Rescore 的区别

```text
function_score 放在主查询：
    对主查询候选直接计算业务评分

function_score 放在 rescore_query：
    只对 Top window_size 候选计算业务评分

rescore：
    是控制昂贵评分只作用于候选窗口的二阶段机制
```

### 23.4 Score Mode 高频点

```text
Rescore score_mode：
total / multiply / avg / max / min

Function Score score_mode：
控制多个 function 如何合并

Function Score boost_mode：
控制 function 分数与原始 query 分数如何合并
```

三者名字相近，但作用层级不同。

## 24. 参考资料

- [Elastic：Rescore search results](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/rescore-search-results)
- [Elastic：Learning To Rank](https://www.elastic.co/docs/solutions/search/ranking/learning-to-rank-search-usage)
- [Elastic：Elasticsearch 5.6 Java API](https://artifacts.elastic.co/javadoc/org/elasticsearch/elasticsearch/5.6.12/index-all.html)
- [Elastic：Multi-match query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-multi-match-query)
- [Elastic：Boolean query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-bool-query)
- [Elastic：Function score query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-function-score-query)
- [Elastic：Constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html)
- [Elastic：Profile search requests](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-profile.html)
- [Elastic：Explain API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-explain)

