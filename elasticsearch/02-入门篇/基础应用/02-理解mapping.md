​    映射（mapping）就像数据库中的 Schema ，描述了文档可能具有的字段或属性、 每个字段的数据类型，比如 text，keyword，integer 或 date ，以及 Lucene 是如何索引和存储这些字段的。



## 核心简单字段类型

​    Elasticsearch 支持如下简单字段类型： 

- 字符串：text、keyword
- 整数：byte、short、integer、long
- 浮点数：float、double
- 布尔型：boolean
- 日期：date



​     更多的字段类型比如 geo_point，ip，nested 等可以在链接处查看：https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html。

​     当你索引一个包含新字段的文档之前，未曾出现 Elasticsearch 会使用动态映射，通过 JSON 中基本数据类型，尝试猜测字段类型，使用如下规则：

 

| JSON数据                     | 字段类型      |
| ---------------------------- | ------------- |
| true或者false                | boolean       |
| 整数：123                    | long          |
| 浮点数：123.45               | double        |
| 字符串，有效日期：2021-05-01 | date          |
| 字符串：foo bar              | text和keyword |

​    **注意**：如果你通过引号 ( "123" ) 索引一个数字，它会被映射为字符串类型 text和 keyword，而不是 long 。但如果这个字段已经映射为 long ，那么 Elasticsearch会尝试将这个字符串转化为 long （在 coerce 设置为 true 的情况下），如果无法转化，则抛出一个异常。

​    

## 查看映射

​    通过 /_mapping ，我们可以查看 Elasticsearch 在一个或多个索引中的映射。 

​    Elasticsearch 文档写入示例：

```shell
PUT news/_doc/1
{
  "title":"冬奥会",
  "pubTime":"2022-02-04T13:12:00",
  "text":"冰雪为媒 共赴冬奥之约"
}
```

查看mapping:

```shell
#查看mapping
GET /news/_mapping

#返回结果
{
  "news" : {
    "mappings" : {
      "properties" : {
        "pubTime" : {
          "type" : "date"
        },
        "text" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```

   Elasticsearch 根据我们索引的文档，为字段动态生成的映射。

   注意：错误的映射，例如将年龄字段映射为 text 类型，而不是 integer ，会导致查询出现令人困惑的结果。



## 自定义字段映射

​    尽管在很多情况下基本字段数据类型已经够用，但你经常需要为单独字段自定义映射，特别是字符串字段。自定义映射允许你执行下面的操作：

- 全文字符串字段和精确值字符串字段的区别 
- 使用特定语言分析器 
- 优化字段以适应部分匹配 
- 指定自定义数据格式 
- 还有更多



   字段最重要的属性是 type 。

```shell
{
  "pubTime" : {
          "type" : "date"
        }
}
```

   字符串字段类型，包括全文字符串 text 和精确值字符串 keyword。

   text 类型字段的最重要属性是分析器 analyzer，默认 Elasticsearch 使用standard 分析器， 但你可以指定一个内置的分析器替代它，例如 whitespace 、 simple 、english、cjk：

 

```shell
{
 "text" : {
          "type" : "text",
          "analyzer" : "cjk"
  }
}
```



## 创建/更新映射

​     当你首次创建一个索引的时候，可以指定类型的映射。你也可以使用 /_mapping更新映射。

​     我们可以更新一个映射来添加一个新字段，但不能更新一个现有的 mapping 把它的字段类型从一个变为另外一个，比如从 text 变为 keyword。我们可以在维持现有mapping 的情况下，把一个字段变成一个 multi-field 字段。

​     为了描述指定映射的两种方式，我们先删除 news索引：

```shell
DELETE news
```

​    创建一个新索引，指定 message 字段使用 cjk 分析器： 

```shell
PUT news
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }, 
  "mappings" : {
      "properties" : {
        "pubTime" : {
          "type" : "date"
        },
        "text" : {
          "type" : "text",
          "analyzer" : "cjk"
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
}
```

​    通过消息体中指定的 mappings 创建了索引映射，索引设置 settings 中通过number_of_shards 指定分片数，number_of_replicas 指定副本数。

   给映射增加一个新的名为 tag 的 keyword 类型字段，使用 _mapping ：

```shell
PUT news/_mapping
{
  "properties" : {
    "tag":{
      "type": "keyword"
    }
  }
}
```



​       我们不需要再次列出所有已存在的字段，因为无论如何我们都无法改变它们，新字段已经被合并到存在的映射中。

​     然后我们查看：

```shell
GET news/_mapping

结果返回：
{
  "news" : {
    "mappings" : {
      "properties" : {
        "pubTime" : {
          "type" : "date"
        },
        "tag" : {
          "type" : "keyword"
        },
        "text" : {
          "type" : "text",
          "analyzer" : "cjk"
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```



## 测试映射

​     可以使用 analyze API 测试字符串字段的映射，比较下面两个请求的输出：

```shell
GET /news/_analyze
{
  "field": "text",
  "text": "搜索索引引擎"
}

GET /news/_analyze
{
  "field": "tag",
  "text": "搜索引擎"
}
```

​    text体里面传入我们想要分析的文本。message 字段产生 3 个词条“搜索”、“索引”和“引擎”， tag 字段产生单独的词条”搜索引擎“，换句话说，我们的映射正常工作。

​    