---
title: Elasticsearch 查询
date: 2022-01-18 08:01:08
order: 05
categories:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
tags:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
  - 查询
permalink: /pages/f8fab8f0/
---

# Elasticsearch 查询

Elasticsearch 查询语句采用基于 RESTful 风格的接口封装成 JSON 格式的对象，称之为 Query DSL。Elasticsearch 查询分类大致分为**全文查询**、**词项查询**、**复合查询**、**嵌套查询**、**位置查询**、**特殊查询**。Elasticsearch 查询从机制分为两种，一种是根据用户输入的查询词，通过排序模型计算文档与查询词之间的**相关度**，并根据评分高低排序返回；另一种是**过滤机制**，只根据过滤条件对文档进行过滤，不计算评分，速度相对较快。

## 全文查询

ES 全文查询主要用于在全文字段上，主要考虑查询词与文档的相关性（Relevance）。

### intervals query

[**`intervals query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html) 根据匹配词的顺序和近似度返回文档。

intervals query 使用**匹配规则**，这些规则应用于指定字段中的 term。

示例：下面示例搜索 `query` 字段，搜索值是 `my favorite food`，没有任何间隙；然后是 `my_text` 字段搜索匹配 `hot water`、`cold porridge` 的 term。

当 my_text 中的值为 `my favorite food is cold porridge` 时，会匹配成功，但是 `when it's cold my favorite food is porridge` 则匹配失败

```bash
POST _search
{
  "query": {
    "intervals" : {
      "my_text" : {
        "all_of" : {
          "ordered" : true,
          "intervals" : [
            {
              "match" : {
                "query" : "my favorite food",
                "max_gaps" : 0,
                "ordered" : true
              }
            },
            {
              "any_of" : {
                "intervals" : [
                  { "match" : { "query" : "hot water" } },
                  { "match" : { "query" : "cold porridge" } }
                ]
              }
            }
          ]
        }
      }
    }
  }
}
```

### match query

[**`match query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) **用于搜索单个字段**，首先会针对查询语句进行解析（经过 analyzer），主要是对查询语句进行分词，分词后查询语句的任何一个词项被匹配，文档就会被搜到，默认情况下相当于对分词后词项进行 or 匹配操作。

[**`match query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) 是执行全文搜索的标准查询，包括模糊匹配选项。

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "George Hubbard"
      }
    }
  }
}
```

等同于 `or` 匹配操作，如下：

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "George Hubbard",
        "operator": "or"
      }
    }
  }
}
```

#### match query 简写

可以通过组合 `<field>` 和 `query` 参数来简化匹配查询语法。

示例：

```bash
GET /_search
{
  "query": {
    "match": {
      "message": "this is a test"
    }
  }
}
```

#### match query 如何工作

匹配查询是布尔类型。这意味着会对提供的文本进行分析，分析过程从提供的文本构造一个布尔查询。 `operator` 参数可以设置为 `or` 或 `and` 来控制布尔子句（默认为 `or`）。可以使用 [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html) 参数设置要匹配的可选 `should` 子句的最小数量。

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "customer_full_name": {
        "query": "George Hubbard",
        "operator": "and"
      }
    }
  }
}
```

可以设置 `analyzer` 来控制哪个分析器将对文本执行分析过程。它默认为字段显式映射定义或默认搜索分析器。

`lenient` 参数可以设置为 `true` 以忽略由数据类型不匹配导致的异常，例如尝试使用文本查询字符串查询数字字段。默认为 `false`。

#### match query 的模糊查询

`fuzziness` 允许基于被查询字段的类型进行模糊匹配。请参阅 [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#fuzziness) 的配置。

在这种情况下可以设置 `prefix_length` 和 `max_expansions` 来控制模糊匹配。如果设置了模糊选项，查询将使用 `top_terms_blended_freqs_${max_expansions}` 作为其重写方法，`fuzzy_rewrite` 参数允许控制查询将如何被重写。

默认情况下允许模糊倒转 (`ab` → `ba`)，但可以通过将 `fuzzy_transpositions` 设置为 `false` 来禁用。

```bash
GET /_search
{
  "query": {
    "match": {
      "message": {
        "query": "this is a testt",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

#### zero terms 查询

如果使用的分析器像 stop 过滤器一样删除查询中的所有标记，则默认行为是不匹配任何文档。可以使用 `zero_terms_query` 选项来改变默认行为，它接受 `none`（默认）和 `all` （相当于 `match_all` 查询）。

```bash
GET /_search
{
  "query": {
    "match": {
      "message": {
        "query": "to be or not to be",
        "operator": "and",
        "zero_terms_query": "all"
      }
    }
  }
}
```

### match_bool_prefix query

[**`match_bool_prefix query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-bool-prefix-query.html) 分析其输入并根据这些词构造一个布尔查询。除了最后一个术语之外的每个术语都用于术语查询。最后一个词用于 `prefix query`。

示例：

```bash
GET /_search
{
  "query": {
    "match_bool_prefix" : {
      "message" : "quick brown f"
    }
  }
}
```

等价于

```bash
GET /_search
{
  "query": {
    "bool" : {
      "should": [
        { "term": { "message": "quick" }},
        { "term": { "message": "brown" }},
        { "prefix": { "message": "f"}}
      ]
    }
  }
}
```

`match_bool_prefix query` 和 `match_phrase_prefix query` 之间的一个重要区别是：`match_phrase_prefix query` 将其 term 匹配为短语，但 `match_bool_prefix query` 可以在任何位置匹配其 term。

上面的示例 `match_bool_prefix query` 查询可以匹配包含 `quick brown fox` 的字段，但它也可以快速匹配 `brown fox`。它还可以匹配包含 `quick`、`brown` 和以 `f` 开头的字段，出现在任何位置。

### match_phrase query

[**`match_phrase query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) 即短语匹配，首先会把 query 内容分词，分词器可以自定义，同时文档还要满足以下两个条件才会被搜索到：

1. **分词后所有词项都要出现在该字段中（相当于 and 操作）**。
2. **字段中的词项顺序要一致**。

例如，有以下 3 个文档，使用 **`match_phrase`** 查询 "How are you"，只有前两个文档会被匹配：

```bash
PUT demo/_create/1
{ "desc": "How are you" }

PUT demo/_create/2
{ "desc": "How are you, Jack?"}

PUT demo/_create/3
{ "desc": "are you"}

GET demo/_search
{
  "query": {
    "match_phrase": {
      "desc": "How are you"
    }
  }
}
```

> 说明：
>
> 一个被认定为和短语 How are you 匹配的文档，必须满足以下这些要求：
>
> - How、 are 和 you 需要全部出现在域中。
> - are 的位置应该比 How 的位置大 1 。
> - you 的位置应该比 How 的位置大 2 。

### match_phrase_prefix query

[**`match_phrase_prefix query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html) 和 [**`match_phrase query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) 类似，只不过 [**`match_phrase_prefix query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html) 最后一个 term 会被作为前缀匹配。

```bash
GET demo/_search
{
  "query": {
    "match_phrase_prefix": {
      "desc": "are yo"
    }
  }
}
```

### multi_match query

[**`multi_match query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html) 是 **`match query`** 的升级，**用于搜索多个字段**。

示例：

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": 34.98,
      "fields": [
        "taxful_total_price",
        "taxless_total_price"
      ]
    }
  }
}
```

**`multi_match query`** 的搜索字段可以使用通配符指定，示例如下：

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": 34.98,
      "fields": [
        "taxful_*",
        "taxless_total_price"
      ]
    }
  }
}
```

同时，也可以用**指数符指定搜索字段的权重**。

示例：指定 taxful_total_price 字段的权重是 taxless_total_price 字段的 3 倍，命令如下：

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": 34.98,
      "fields": [
        "taxful_total_price^3",
        "taxless_total_price"
      ]
    }
  }
}
```

### combined_fields query

[**`combined_fields query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-combined-fields-query.html) 支持搜索多个文本字段，就好像它们的内容已被索引到一个组合字段中一样。该查询会生成以 term 为中心的输入字符串视图：首先它将查询字符串解析为独立的 term，然后在所有字段中查找每个 term。当匹配结果可能跨越多个文本字段时，此查询特别有用，例如文章的标题、摘要和正文：

```bash
GET /_search
{
  "query": {
    "combined_fields" : {
      "query":      "database systems",
      "fields":     [ "title", "abstract", "body"],
      "operator":   "and"
    }
  }
}
```

#### 字段前缀权重

字段前缀权重根据组合字段模型进行计算。例如，如果 title 字段的权重为 2，则匹配度打分时会将 title 中的每个 term 形成的组合字段，按出现两次进行打分。

### common_terms query

> 7.3.0 废弃

[**`common_terms query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-common-terms-query.html) 是一种在不牺牲性能的情况下替代停用词提高搜索准确率和召回率的方案。

查询中的每个词项都有一定的代价，以搜索“The brown fox”为例，query 会被解析成三个词项“the”“brown”和“fox”，每个词项都会到索引中执行一次查询。很显然包含“the”的文档非常多，相比其他词项，“the”的重要性会低很多。传统的解决方案是把“the”当作停用词处理，去除停用词之后可以减少索引大小，同时在搜索时减少对停用词的收缩。

虽然停用词对文档评分影响不大，但是当停用词仍然有重要意义的时候，去除停用词就不是完美的解决方案了。如果去除停用词，就无法区分“happy”和“not happy”, “The”“To be or not to be”就不会在索引中存在，搜索的准确率和召回率就会降低。

common_terms query 提供了一种解决方案，它把 query 分词后的词项分成重要词项（低频词项）和不重要的词项（高频词，也就是之前的停用词）。在搜索的时候，首先搜索和重要词项匹配的文档，这些文档是词项出现较少并且词项对其评分影响较大的文档。然后执行第二次查询，搜索对评分影响较小的高频词项，但是不计算所有文档的评分，而是只计算第一次查询已经匹配的文档得分。如果一个查询中只包含高频词，那么会通过 and 连接符执行一个单独的查询，换言之，会搜索所有的词项。

词项是高频词还是低频词是通过 cutoff frequency 来设置阀值的，取值可以是绝对频率（频率大于 1）或者相对频率（0 ～ 1）。common_terms query 最有趣之处在于它能自适应特定领域的停用词，例如，在视频托管网站上，诸如“clip”或“video”之类的高频词项将自动表现为停用词，无须保留手动列表。

例如，文档频率高于 0.1% 的词项将会被当作高频词项，词频之间可以用 low_freq_operator、high_freq_operator 参数连接。设置低频词操作符为“and”使所有的低频词都是必须搜索的，示例代码如下：

```bash
GET books/_search
{
	"query": {
		"common": {
			"body": {
				"query": "nelly the elephant as a cartoon",
				"cutoff_frequency": 0.001,
				"low_freq_operator": "and"
			}
		}
	}
}
```

上述操作等价于：

```bash
GET books/_search
{
	"query": {
		"bool": {
			"must": [
			  { "term": { "body": "nelly" } },
			  { "term": { "body": "elephant" } },
			  { "term": { "body": "cartoon" } }
			],
			"should": [
			  { "term": { "body": "the" } },
			  { "term": { "body": "as" } },
			  { "term": { "body": "a" } }
			]
		}
	}
}
```

### query_string query

[**`query_string query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html) 是与 Lucene 查询语句的语法结合非常紧密的一种查询，允许在一个查询语句中使用多个特殊条件关键字（如：AND | OR | NOT）对多个字段进行查询，建议熟悉 Lucene 查询语法的用户去使用。

用户可以使用 query_string query 来创建包含通配符、跨多个字段的搜索等复杂搜索。虽然通用，但查询是严格的，如果查询字符串包含任何无效语法，则会返回错误。

示例：

```bash
GET /_search
{
  "query": {
    "query_string": {
      "query": "(new york city) OR (big apple)",
      "default_field": "content"
    }
  }
}
```

### simple_query_string query

[**`simple_query_string query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html) 是一种适合直接暴露给用户，并且具有非常完善的查询语法的查询语句，接受 Lucene 查询语法，解析过程中发生错误不会抛出异常。

虽然语法比 [**`query_string query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html) 更严格，但 [**`simple_query_string query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html) 不会返回无效语法的错误。相反，它会忽略查询字符串的任何无效部分。

示例：

```bash
GET /_search
{
  "query": {
    "simple_query_string" : {
        "query": "\"fried eggs\" +(eggplant | potato) -frittata",
        "fields": ["title^5", "body"],
        "default_operator": "and"
    }
  }
}
```

#### simple_query_string 语义

- `+`：等价于 AND 操作
- `|`：等价于 OR 操作
- `-`：相当于 NOT 操作
- `"`：包装一些标记以表示用于搜索的短语
- `*`：词尾表示前缀查询
- `(` and `)`：表示优先级
- `~N`：词尾表示表示编辑距离（模糊性）
- `~N`：在一个短语之后表示溢出量

注意：要使用上面的字符，请使用反斜杠 `/` 对其进行转义。

### 全文查询完整示例

```bash
#设置 position_increment_gap
DELETE groups
PUT groups
{
  "mappings": {
    "properties": {
      "names":{
        "type": "text",
        "position_increment_gap": 0
      }
    }
  }
}

GET groups/_mapping

POST groups/_doc
{
  "names": [ "John Water", "Water Smith"]
}

POST groups/_search
{
  "query": {
    "match_phrase": {
      "names": {
        "query": "Water Water",
        "slop": 100
      }
    }
  }
}

POST groups/_search
{
  "query": {
    "match_phrase": {
      "names": "Water Smith"
    }
  }
}

DELETE groups
```

## 词项查询

**`Term`（词项）是表达语意的最小单位**。搜索和利用统计语言模型进行自然语言处理都需要处理 Term。

全文查询在执行查询之前会分析查询字符串。

与全文查询不同，词项查询不会分词，而是将输入作为一个整体，在倒排索引中查找准确的词项。并且使用相关度计算公式为每个包含该词项的文档进行相关度计算。一言以概之：**词项查询是对词项进行精确匹配**。词项查询通常用于结构化数据，如数字、日期和枚举类型。

词项查询有以下类型：

- **[`exists` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html)**
- **[`fuzzy` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)**
- **[`ids` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-ids-query.html)**
- **[`prefix` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html)**
- **[`range` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html)**
- **[`regexp` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html)**
- **[`term` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html)**
- **[`terms` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html)**
- **[`type` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-type-query.html)**
- **[`wildcard` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html)**

### exists query

[**`exists query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html) 会返回字段中至少有一个非空值的文档。

由于多种原因，文档字段可能不存在索引值：

- JSON 中的字段为 `null` 或 `[]`
- 该字段在 mapping 中配置了 `"index" : false`
- 字段值的长度超过了 mapping 中的 `ignore_above` 设置
- 字段值格式错误，并且在 mapping 中定义了 `ignore_malformed`

示例：

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "exists": {
      "field": "email"
    }
  }
}
```

以下文档会匹配上面的查询：

- `{ "user" : "jane" }` 有 user 字段，且不为空。
- `{ "user" : "" }` 有 user 字段，值为空字符串。
- `{ "user" : "-" }` 有 user 字段，值不为空。
- `{ "user" : [ "jane" ] }` 有 user 字段，值不为空。
- `{ "user" : [ "jane", null ] }` 有 user 字段，至少一个值不为空即可。

下面的文档都不会被匹配：

- `{ "user" : null }` 虽然有 user 字段，但是值为空。
- `{ "user" : [] }` 虽然有 user 字段，但是值为空。
- `{ "user" : [null] }` 虽然有 user 字段，但是值为空。
- `{ "foo" : "bar" }` 没有 user 字段。

### fuzzy query

[**`fuzzy query`**（模糊查询）](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)返回包含与搜索词相似的词的文档。ES 使用 [Levenshtein edit distance（Levenshtein 编辑距离）](https://en.wikipedia.org/wiki/Levenshtein_distance)测量相似度或模糊度。

编辑距离是将一个术语转换为另一个术语所需的单个字符更改的数量。这些变化可能包括：

- 改变一个字符：（**b**ox -> **f**ox）
- 删除一个字符：（**b**lack -> lack）
- 插入一个字符：（sic -> sic**k**）
- 反转两个相邻字符：（**ac**t → **ca**t）

为了找到相似的词条，fuzzy query 会在指定的编辑距离内创建搜索词条的所有可能变体或扩展集。然后返回完全匹配任意扩展的文档。

```bash
GET books/_search
{
  "query": {
    "fuzzy": {
      "user.id": {
        "value": "ki",
        "fuzziness": "AUTO",
        "max_expansions": 50,
        "prefix_length": 0,
        "transpositions": true,
        "rewrite": "constant_score"
      }
    }
  }
}
```

注意：如果配置了 [`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) ，则 fuzzy query 不能执行。

### ids query

[**`ids query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-ids-query.html) 根据 ID 返回文档。 此查询使用存储在 `_id` 字段中的文档 ID。

```bash
GET /_search
{
  "query": {
    "ids" : {
      "values" : ["1", "4", "100"]
    }
  }
}
```

### prefix query

[**`prefix query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html#prefix-query-ex-request) 用于查询某个字段中包含指定前缀的文档。

比如查询 `user.id` 中含有以 `ki` 为前缀的关键词的文档，那么含有 `kind`、`kid` 等所有以 `ki` 开头关键词的文档都会被匹配。

```bash
GET /_search
{
  "query": {
    "prefix": {
      "user.id": {
        "value": "ki"
      }
    }
  }
}
```

### range query

[**`range query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) 即范围查询，用于匹配在某一范围内的数值型、日期类型或者字符串型字段的文档。比如搜索哪些书籍的价格在 50 到 100 之间、哪些书籍的出版时间在 2015 年到 2019 年之间。**使用 range 查询只能查询一个字段，不能作用在多个字段上**。

range 查询支持的参数有以下几种：

- **`gt`**：大于

- **`gte`**：大于等于

- **`lt`**：小于

- **`lte`**：小于等于

- **`format`**：如果字段是 Date 类型，可以设置日期格式化

- **`time_zone`**：时区

- **`relation`**：指示范围查询如何匹配范围字段的值。

  - **`INTERSECTS` (Default)**：匹配与查询字段值范围相交的文档。
  - **`CONTAINS`**：匹配完全包含查询字段值的文档。
  - **`WITHIN`**：匹配具有完全在查询范围内的范围字段值的文档。

示例：数值范围查询

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "taxful_total_price": {
        "gt": 10,
        "lte": 50
      }
    }
  }
}
```

示例：日期范围查询

```bash
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "order_date": {
        "time_zone": "+00:00",
        "gte": "2018-01-01T00:00:00",
        "lte": "now"
      }
    }
  }
}
```

### regexp query

[**`regexp query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html) 返回与正则表达式相匹配的 term 所属的文档。

[正则表达式](https://zh.wikipedia.org/zh-hans/%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F)是一种使用占位符字符匹配数据模式的方法，称为运算符。

示例：以下搜索返回 `user.id` 字段包含任何以 `k` 开头并以 `y` 结尾的文档。 `.*` 运算符匹配任何长度的任何字符，包括无字符。匹配项可以包括 `ky`、`kay` 和 `kimchy`。

```bash
GET /_search
{
  "query": {
    "regexp": {
      "user.id": {
        "value": "k.*y",
        "flags": "ALL",
        "case_insensitive": true,
        "max_determinized_states": 10000,
        "rewrite": "constant_score"
      }
    }
  }
}
```

> 注意：如果配置了[`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) ，则 [**`regexp query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html) 会被禁用。

### term query

[**`term query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html) 用来查找指定字段中包含给定单词的文档，term 查询不被解析，只有查询词和文档中的词精确匹配才会被搜索到，应用场景为查询人名、地名等需要精准匹配的需求。

示例：

```bash
# 1. 创建一个索引
DELETE my-index-000001
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "full_text": { "type": "text" }
    }
  }
}

# 2. 使用 "Quick Brown Foxes!" 关键字查 "full_text" 字段
PUT my-index-000001/_doc/1
{
  "full_text": "Quick Brown Foxes!"
}

# 3. 使用 term 查询
GET my-index-000001/_search?pretty
{
  "query": {
    "term": {
      "full_text": "Quick Brown Foxes!"
    }
  }
}
# 因为 full_text 字段不再包含确切的 Term —— "Quick Brown Foxes!"，所以 term query 搜索不到任何结果

# 4. 使用 match 查询
GET my-index-000001/_search?pretty
{
  "query": {
    "match": {
      "full_text": "Quick Brown Foxes!"
    }
  }
}

DELETE my-index-000001
```

> :warning: 注意：应避免 term 查询对 text 字段使用查询。
>
> 默认情况下，Elasticsearch 针对 text 字段的值进行解析分词，这会使查找 text 字段值的精确匹配变得困难。
>
> 要搜索 text 字段值，需改用 match 查询。

### terms query

[**`terms query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html) 与 [**`term query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html) 相同，但可以搜索多个值。

terms query 查询参数：

- **`index`**：索引名
- **`id`**：文档 ID
- **`path`**：要从中获取字段值的字段的名称，即搜索关键字
- **`routing`**（选填）：要从中获取 term 值的文档的自定义路由值。如果在索引文档时提供了自定义路由值，则此参数是必需的。

示例：

```bash
# 1. 创建一个索引
DELETE my-index-000001
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "color": { "type": "keyword" }
    }
  }
}

# 2. 写入一个文档
PUT my-index-000001/_doc/1
{
  "color": [
    "blue",
    "green"
  ]
}

# 3. 写入另一个文档
PUT my-index-000001/_doc/2
{
  "color": "blue"
}

# 3. 使用 terms query
GET my-index-000001/_search?pretty
{
  "query": {
    "terms": {
      "color": {
        "index": "my-index-000001",
        "id": "2",
        "path": "color"
      }
    }
  }
}

DELETE my-index-000001
```

### type query

> 7.0.0 后废弃

[**`type query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-type-query.html) 用于查询具有指定类型的文档。

示例：

```bash
GET /_search
{
  "query": {
    "type": {
      "value": "_doc"
    }
  }
}
```

### wildcard query

[**`wildcard query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html) 即通配符查询，返回与通配符模式匹配的文档。

`?` 用来匹配一个任意字符，`*` 用来匹配零个或者多个字符。

示例：以下搜索返回 `user.id` 字段包含以 `ki` 开头并以 `y` 结尾的术语的文档。这些匹配项可以包括 `kiy`、`kity` 或 `kimchy`。

```bash
GET /_search
{
  "query": {
    "wildcard": {
      "user.id": {
        "value": "ki*y",
        "boost": 1.0,
        "rewrite": "constant_score"
      }
    }
  }
}
```

> 注意：如果配置了[`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html#query-dsl-allow-expensive-queries) ，则[**`wildcard query`**](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html) 会被禁用。

### 词项查询完整示例

```bash
DELETE products
PUT products
{
  "settings": {
    "number_of_shards": 1
  }
}

POST /products/_bulk
{ "index": { "_id": 1 }}
{ "productID" : "XHDK-A-1293-#fJ3","desc":"iPhone" }
{ "index": { "_id": 2 }}
{ "productID" : "KDKE-B-9947-#kL5","desc":"iPad" }
{ "index": { "_id": 3 }}
{ "productID" : "JODL-X-1937-#pV7","desc":"MBP" }

GET /products

POST /products/_search
{
  "query": {
    "term": {
      "desc": {
        //"value": "iPhone"
        "value":"iphone"
      }
    }
  }
}

POST /products/_search
{
  "query": {
    "term": {
      "desc.keyword": {
        //"value": "iPhone"
        //"value":"iphone"
      }
    }
  }
}

POST /products/_search
{
  "query": {
    "term": {
      "productID": {
        "value": "XHDK-A-1293-#fJ3"
      }
    }
  }
}

POST /products/_search
{
  //"explain": true,
  "query": {
    "term": {
      "productID.keyword": {
        "value": "XHDK-A-1293-#fJ3"
      }
    }
  }
}

POST /products/_search
{
  "explain": true,
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "productID.keyword": "XHDK-A-1293-#fJ3"
        }
      }

    }
  }
}
```

## 复合查询

复合查询就是把一些简单查询组合在一起实现更复杂的查询需求，除此之外，复合查询还可以控制另外一个查询的行为。

### bool query

bool 查询可以把任意多个简单查询组合在一起，使用 must、should、must_not、filter 选项来表示简单查询之间的逻辑，每个选项都可以出现 0 次到多次，它们的含义如下：

- must 文档必须匹配 must 选项下的查询条件，相当于逻辑运算的 AND，且参与文档相关度的评分。
- should 文档可以匹配 should 选项下的查询条件也可以不匹配，相当于逻辑运算的 OR，且参与文档相关度的评分。
- must_not 与 must 相反，匹配该选项下的查询条件的文档不会被返回；需要注意的是，**must_not 语句不会影响评分，它的作用只是将不相关的文档排除**。
- filter 和 must 一样，匹配 filter 选项下的查询条件的文档才会被返回，**但是 filter 不评分，只起到过滤功能，与 must_not 相反**。

假设要查询 title 中包含关键词 java，并且 price 不能高于 70，description 可以包含也可以不包含虚拟机的书籍，构造 bool 查询语句如下：

```
GET books/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": 1
        }
      },
      "must_not": {
        "range": {
          "price": {
            "gte": 70
          }
        }
      },
      "must": {
        "match": {
          "title": "java"
        }
      },
      "should": [
        {
          "match": {
            "description": "虚拟机"
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

有关布尔查询更详细的信息参考 [bool query（组合查询）详解](https://www.knowledgedict.com/tutorial/elasticsearch-query-bool.html)。

### boosting query

boosting 查询用于需要对两个查询的评分进行调整的场景，boosting 查询会把两个查询封装在一起并降低其中一个查询的评分。

boosting 查询包括 positive、negative 和 negative_boost 三个部分，positive 中的查询评分保持不变，negative 中的查询会降低文档评分，negative_boost 指明 negative 中降低的权值。如果我们想对 2015 年之前出版的书降低评分，可以构造一个 boosting 查询，查询语句如下：

```
GET books/_search
{
	"query": {
		"boosting": {
			"positive": {
				"match": {
					"title": "python"
				}
			},
			"negative": {
				"range": {
					"publish_time": {
						"lte": "2015-01-01"
					}
				}
			},
			"negative_boost": 0.2
		}
	}
}
```

boosting 查询中指定了抑制因子为 0.2，publish_time 的值在 2015-01-01 之后的文档得分不变，publish_time 的值在 2015-01-01 之前的文档得分为原得分的 0.2 倍。

### constant_score query

constant*score query 包装一个 filter query，并返回匹配过滤器查询条件的文档，且它们的相关性评分都等于 \_boost* 参数值（可以理解为原有的基于 tf-idf 或 bm25 的相关分固定为 1.0，所以最终评分为 _1.0 \* boost_，即等于 _boost_ 参数值）。下面的查询语句会返回 title 字段中含有关键词 _elasticsearch_ 的文档，所有文档的评分都是 1.8：

```
GET books/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "title": "elasticsearch"
        }
      },
      "boost": 1.8
    }
  }
}
```

### dis_max query

dis_max query 与 bool query 有一定联系也有一定区别，dis_max query 支持多并发查询，可返回与任意查询条件子句匹配的任何文档类型。与 bool 查询可以将所有匹配查询的分数相结合使用的方式不同，dis_max 查询只使用最佳匹配查询条件的分数。请看下面的例子：

```
GET books/_search
{
	"query": {
		"dis_max": {
			"tie_breaker": 0.7,
			"boost": 1.2,
			"queries": [{
					"term": {
						"age": 34
					}
				},
				{
					"term": {
						"age": 35
					}
				}
			]
		}
	}
}
```

### function_score query

function_score query 可以修改查询的文档得分，这个查询在有些情况下非常有用，比如通过评分函数计算文档得分代价较高，可以改用过滤器加自定义评分函数的方式来取代传统的评分方式。

使用 function_score query，用户需要定义一个查询和一至多个评分函数，评分函数会对查询到的每个文档分别计算得分。

下面这条查询语句会返回 books 索引中的所有文档，文档的最大得分为 5，每个文档的得分随机生成，权重的计算模式为相乘模式。

```
GET books/_search
{
  "query": {
    "function_score": {
      "query": {
        "match all": {}
      },
      "boost": "5",
      "random_score": {},
      "boost_mode": "multiply"
    }
  }
}
```

使用脚本自定义评分公式，这里把 price 值的十分之一开方作为每个文档的得分，查询语句如下：

```
GET books/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "script_score": {
        "inline": "Math.sqrt(doc['price'].value/10)"
      }
    }
  }
}
```

关于 function_score 的更多详细内容请查看 [Elasticsearch function_score 查询最强详解](https://www.knowledgedict.com/tutorial/elasticsearch-function_score.html)。

### indices query

indices query 适用于需要在多个索引之间进行查询的场景，它允许指定一个索引名字列表和内部查询。indices query 中有 query 和 no_match_query 两部分，query 中用于搜索指定索引列表中的文档，no_match_query 中的查询条件用于搜索指定索引列表之外的文档。下面的查询语句实现了搜索索引 books、books2 中 title 字段包含关键字 javascript，其他索引中 title 字段包含 basketball 的文档，查询语句如下：

```
GET books/_search
{
	"query": {
		"indices": {
			"indices": ["books", "books2"],
			"query": {
				"match": {
					"title": "javascript"
				}
			},
			"no_match_query": {
				"term": {
					"title": "basketball"
				}
			}
		}
	}
}
```

## 嵌套查询

在 Elasticsearch 这样的分布式系统中执行全 SQL 风格的连接查询代价昂贵，是不可行的。相应地，为了实现水平规模地扩展，Elasticsearch 提供了以下两种形式的 join：

- nested query（嵌套查询）

  文档中可能包含嵌套类型的字段，这些字段用来索引一些数组对象，每个对象都可以作为一条独立的文档被查询出来。

- has_child query（有子查询）和 has_parent query（有父查询）

  父子关系可以存在单个的索引的两个类型的文档之间。has_child 查询将返回其子文档能满足特定查询的父文档，而 has_parent 则返回其父文档能满足特定查询的子文档。

### nested query

文档中可能包含嵌套类型的字段，这些字段用来索引一些数组对象，每个对象都可以作为一条独立的文档被查询出来（用嵌套查询）。

```
PUT /my_index
{
	"mappings": {
		"type1": {
			"properties": {
				"obj1": {
					"type": "nested"
				}
			}
		}
	}
}
```

### has_child query

文档的父子关系创建索引时在映射中声明，这里以员工（employee）和工作城市（branch）为例，它们属于不同的类型，相当于数据库中的两张表，如果想把员工和他们工作的城市关联起来，需要告诉 Elasticsearch 文档之间的父子关系，这里 employee 是 child type，branch 是 parent type，在映射中声明，执行命令：

```
PUT /company
{
	"mappings": {
		"branch": {},
		"employee": {
			"parent": { "type": "branch" }
		}
	}
}
```

使用 bulk api 索引 branch 类型下的文档，命令如下：

```
POST company/branch/_bulk
{ "index": { "_id": "london" }}
{ "name": "London Westminster","city": "London","country": "UK" }
{ "index": { "_id": "liverpool" }}
{ "name": "Liverpool Central","city": "Liverpool","country": "UK" }
{ "index": { "_id": "paris" }}
{ "name": "Champs Elysees","city": "Paris","country": "France" }
```

添加员工数据：

```
POST company/employee/_bulk
{ "index": { "_id": 1,"parent":"london" }}
{ "name": "Alice Smith","dob": "1970-10-24","hobby": "hiking" }
{ "index": { "_id": 2,"parent":"london" }}
{ "name": "Mark Tomas","dob": "1982-05-16","hobby": "diving" }
{ "index": { "_id": 3,"parent":"liverpool" }}
{ "name": "Barry Smith","dob": "1979-04-01","hobby": "hiking" }
{ "index": { "_id": 4,"parent":"paris" }}
{ "name": "Adrien Grand","dob": "1987-05-11","hobby": "horses" }
```

通过子文档查询父文档要使用 has_child 查询。例如，搜索 1980 年以后出生的员工所在的分支机构，employee 中 1980 年以后出生的有 Mark Thomas 和 Adrien Grand，他们分别在 london 和 paris，执行以下查询命令进行验证：

```
GET company/branch/_search
{
	"query": {
		"has_child": {
			"type": "employee",
			"query": {
				"range": { "dob": { "gte": "1980-01-01" } }
			}
		}
	}
}
```

搜索哪些机构中有名为 “Alice Smith” 的员工，因为使用 match 查询，会解析为 “Alice” 和 “Smith”，所以 Alice Smith 和 Barry Smith 所在的机构会被匹配，执行以下查询命令进行验证：

```
GET company/branch/_search
{
	"query": {
		"has_child": {
			"type": "employee",
			"score_mode": "max",
			"query": {
				"match": { "name": "Alice Smith" }
			}
		}
	}
}
```

可以使用 min_children 指定子文档的最小个数。例如，搜索最少含有两个 employee 的机构，查询命令如下：

```
GET company/branch/_search?pretty
{
	"query": {
		"has_child": {
			"type": "employee",
			"min_children": 2,
			"query": {
				"match_all": {}
			}
		}
	}
}
```

### has_parent query

通过父文档查询子文档使用 has_parent 查询。比如，搜索哪些 employee 工作在 UK，查询命令如下：

```
GET company/employee/_search
{
	"query": {
		"has_parent": {
			"parent_type": "branch",
			"query": {
				"match": { "country": "UK }
			}
		}
	}
}
```

## 位置查询

Elasticsearch 可以对地理位置点 geo_point 类型和地理位置形状 geo_shape 类型的数据进行搜索。为了学习方便，这里准备一些城市的地理坐标作为测试数据，每一条文档都包含城市名称和地理坐标这两个字段，这里的坐标点取的是各个城市中心的一个位置。首先把下面的内容保存到 geo.json 文件中：

```
{"index":{ "_index":"geo","_type":"city","_id":"1" }}
{"name":"北京","location":"39.9088145109,116.3973999023"}
{"index":{ "_index":"geo","_type":"city","_id": "2" }}
{"name":"乌鲁木齐","location":"43.8266300000,87.6168800000"}
{"index":{ "_index":"geo","_type":"city","_id":"3" }}
{"name":"西安","location":"34.3412700000,108.9398400000"}
{"index":{ "_index":"geo","_type":"city","_id":"4" }}
{"name":"郑州","location":"34.7447157466,113.6587142944"}
{"index":{ "_index":"geo","_type":"city","_id":"5" }}
{"name":"杭州","location":"30.2294080260,120.1492309570"}
{"index":{ "_index":"geo","_type":"city","_id":"6" }}
{"name":"济南","location":"36.6518400000,117.1200900000"}
```

创建一个索引并设置映射：

```
PUT geo
{
	"mappings": {
		"city": {
			"properties": {
				"name": {
					"type": "keyword"
				},
				"location": {
					"type": "geo_point"
				}
			}
		}
	}
}
```

然后执行批量导入命令：

```
curl -XPOST "http://localhost:9200/_bulk?pretty" --data-binary @geo.json
```

### geo_distance query

geo_distance query 可以查找在一个中心点指定范围内的地理点文档。例如，查找距离天津 200km 以内的城市，搜索结果中会返回北京，命令如下：

```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_distance": {
					"distance": "200km",
					"location": {
						"lat": 39.0851000000,
						"lon": 117.1993700000
					}
				}
			}
		}
	}
}
```

按各城市离北京的距离排序：

```
GET geo/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [{
    "_geo_distance": {
      "location": "39.9088145109,116.3973999023",
      "unit": "km",
      "order": "asc",
      "distance_type": "plane"
    }
  }]
}
```

其中 location 对应的经纬度字段；unit 为 `km` 表示将距离以 `km` 为单位写入到每个返回结果的 sort 键中；distance_type 为 `plane` 表示使用快速但精度略差的 `plane` 计算方式。

### geo_bounding_box query

geo_bounding_box query 用于查找落入指定的矩形内的地理坐标。查询中由两个点确定一个矩形，然后在矩形区域内查询匹配的文档。

```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_bounding_box": {
					"location": {
						"top_left": {
							"lat": 38.4864400000,
							"lon": 106.2324800000
						},
						"bottom_right": {
							"lat": 28.6820200000,
							"lon": 115.8579400000
						}
					}
				}
			}
		}
	}
}
```

### geo_polygon query

geo_polygon query 用于查找在指定**多边形**内的地理点。例如，呼和浩特、重庆、上海三地组成一个三角形，查询位置在该三角形区域内的城市，命令如下：

```
GET geo/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			}
		},
		"filter": {
			"geo_polygon": {
				"location": {
					"points": [{
						"lat": 40.8414900000,
						"lon": 111.7519900000
					}, {
						"lat": 29.5647100000,
						"lon": 106.5507300000
					}, {
						"lat": 31.2303700000,
						"lon": 121.4737000000
					}]
				}
			}
		}
	}
}
```

### geo_shape query

geo_shape query 用于查询 geo_shape 类型的地理数据，地理形状之间的关系有相交、包含、不相交三种。创建一个新的索引用于测试，其中 location 字段的类型设为 geo_shape 类型。

```
PUT geoshape
{
	"mappings": {
		"city": {
			"properties": {
				"name": {
					"type": "keyword"
				},
				"location": {
					"type": "geo_shape"
				}
			}
		}
	}
}
```

关于经纬度的顺序这里做一个说明，geo_point 类型的字段纬度在前经度在后，但是对于 geo_shape 类型中的点，是经度在前纬度在后，这一点需要特别注意。

把西安和郑州连成的线写入索引：

```
POST geoshape/city/1
{
	"name": "西安-郑州",
	"location": {
		"type": "linestring",
		"coordinates": [
			[108.9398400000, 34.3412700000],
			[113.6587142944, 34.7447157466]
		]
	}
}
```

查询包含在由银川和南昌作为对角线上的点组成的矩形的地理形状，由于西安和郑州组成的直线落在该矩形区域内，因此可以被查询到。命令如下：

```
GET geoshape/_search
{
	"query": {
		"bool": {
			"must": {
				"match_all": {}
			},
			"filter": {
				"geo_shape": {
					"location": {
						"shape": {
							"type": "envelope",
							"coordinates": [
								[106.23248, 38.48644],
								[115.85794, 28.68202]
							]
						},
						"relation": "within"
					}
				}
			}
		}
	}
}
```

## 特殊查询

### more_like_this query

more_like_this query 可以查询和提供文本类似的文档，通常用于近似文本的推荐等场景。查询命令如下：

```
GET books/_search
{
	"query": {
		"more_like_ this": {
			"fields": ["title", "description"],
			"like": "java virtual machine",
			"min_term_freq": 1,
			"max_query_terms": 12
		}
	}
}
```

可选的参数及取值说明如下：

- fields 要匹配的字段，默认是 \_all 字段。
- like 要匹配的文本。
- min_term_freq 文档中词项的最低频率，默认是 2，低于此频率的文档会被忽略。
- max_query_terms query 中能包含的最大词项数目，默认为 25。
- min_doc_freq 最小的文档频率，默认为 5。
- max_doc_freq 最大文档频率。
- min_word length 单词的最小长度。
- max_word length 单词的最大长度。
- stop_words 停用词列表。
- analyzer 分词器。
- minimum_should_match 文档应匹配的最小词项数，默认为 query 分词后词项数的 30%。
- boost terms 词项的权重。
- include 是否把输入文档作为结果返回。
- boost 整个 query 的权重，默认为 1.0。

### script query

Elasticsearch 支持使用脚本进行查询。例如，查询价格大于 180 的文档，命令如下：

```
GET books/_search
{
  "query": {
    "script": {
      "script": {
        "inline": "doc['price'].value > 180",
        "lang": "painless"
      }
    }
  }
}
```

### percolate query

一般情况下，我们是先把文档写入到 Elasticsearch 中，通过查询语句对文档进行搜索。percolate query 则是反其道而行之的做法，它会先注册查询条件，根据文档来查询 query。例如，在 my-index 索引中有一个 laptop 类型，文档有 price 和 name 两个字段，在映射中声明一个 percolator 类型的 query，命令如下：

```
PUT my-index
{
	"mappings": {
		"laptop": {
			"properties": {
				"price": { "type": "long" },
				"name": { "type": "text" }
			},
			"queries": {
				"properties": {
					"query": { "type": "percolator" }
				}
			}
		}
	}
}
```

注册一个 bool query，bool query 中包含一个 range query，要求 price 字段的取值小于等于 10000，并且 name 字段中含有关键词 macbook：

```
PUT /my-index/queries/1?refresh
{
	"query": {
		"bool": {
			"must": [{
				"range": { "price": { "lte": 10000 } }
			}, {
				"match": { "name": "macbook" }
			}]
		}
	}
}
```

通过文档查询 query：

```
GET /my-index/_search
{
	"query": {
		"percolate": {
			"field": "query",
			"document_type": "laptop",
			"document": {
				"price": 9999,
				"name": "macbook pro on sale"
			}
		}
	}
}
```

文档符合 query 中的条件，返回结果中可以查到上文中注册的 bool query。percolate query 的这种特性适用于数据分类、数据路由、事件监控和预警等场景。