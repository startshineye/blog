## **倒排索引**

​       Elasticsearch 使用一种称为倒排索引的结构，它适用于快速的全文搜索。一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。

​      假设我们有两个文档，每个文档的正文字段包含如下内容：

- The quick brown fox jumped over the lazy dog 

- Quick brown foxes leap over lazy dogs in summer

   为了创建倒排索引，我们首先将每个文档的正文字段，拆分成单独的词（我们称它为词条或 Tokens），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。如下所示：

| 词条   | 文档1 | 文档2 |
| ------ | ----- | ----- |
| The    | x     |       |
| quick  | x     |       |
| brown  | x     | x     |
| fox    | x     |       |
| jumped | x     |       |
| over   | x     | x     |
| the    | x     |       |
| lazy   | x     | x     |
| dog    | x     |       |
| Quick  |       | x     |
| foxes  |       | x     |
| leap   |       | x     |
| dogs   |       | x     |
| in     |       | x     |
| summer |       | x     |

​    现在，如果我们想搜索 quick brown，我们只需要查找包含每个词条的文档：
| 词条  | 文档1 | 文档2 |
| ----- | ----- | ----- |
| brown | x     | x     |
| quick | x     |       |

   两个文档都匹配，但是第一个文档比第二个匹配度更高。如果我们使用，仅计算匹配词条数量的简单相似性算法，那么我们可以说，对于我们查询的相关性来讲，第一个文档比第二个文档更佳。 

   但是，我们目前的倒排索引有一些问题：   

- Quick 和 quick 以独立的词条出现，然而用户可能认为它们是相同的词。
- fox 和 foxes 非常相似，就像 dog 和 dogs，他们有相同的词根。
- jumped 和 leap，尽管没有相同的词根，但他们是同义词。



   使用前面的索引搜索 + Quick + fox 不会得到任何匹配文档。（记住，+ 前缀表明这个词必须存在）只有同时出现 Quick 和 fox 的文档才满足这个查询条件，但是第一个文档包含 quick fox，第二个文档包含 Quick foxes。

  我们的用户可以合理的期望两个文档与查询匹配，我们可以做的更好。

  如果我们将词条规范为标准模式，那么我们可以找到与用户搜索的词条不完全一致，但具有足够相关性的文档，例如：

- Quick 可以小写化为 quick 
- foxes 可以词干提取 -- 变为词根的格式 -- 为 fox。类似的，dogs 可以为提取为dog 
- jumped 和 leap 是同义词，可以索引为相同的单词 jump



​    现在索引看上去像这样：
| 词条   | 文档1 | 文档2 |
| ------ | ----- | ----- |
| the    | x     |       |
| quick  | x     | x     |
| brown  | x     | x     |
| fox    | x     | x     |
| jumped | x     | x     |
| over   | x     | x     |
| lazy   | x     | x     |
| dog    | x     | x     |
| in     |       | x     |
| summer |       | x     |

​    这还远远不够。我们搜索 +Quick +fox 仍然会失败，因为在我们的索引中，已经没有 Quick 了。

​    但是，如果我们对**搜索的字符串**，使用与**正文字段相同的标准化规则**，会变成查询 +quick +fox，这样两个文档都会匹配。



## 禁用索引

​     默认情况下，Elasticsearch 文档每个字段都会被索引。如果某些字段不需要支持查询，可以在映射中配置 "index": false ，减少存储空间占用，并且提升写入速度。尽管这个字段不能被搜索，但是它并不妨碍做聚合（如果该字段是可以聚合的字段）。

​    例如，文章的标题、正文、发布时间字段，需要创建索引，文章的 url 字段不需要被索引，创建索引映射时可以按以下方式禁用它：

```
PUT news
{
    "settings":{
        "number_of_shards":5,
        "number_of_replicas":1
    },
    "mappings":{
        "properties":{
            "title":{
                "type":"text",
                "analyzer":"ik_max_word"
            },
            "content":{
                "type":"text",
                "analyzer":"ik_max_word"
            },
            "createtime":{
                "type":"date"
            },
            "url":{
                "type":"keyword",
                "index":false
            }
        }
    }
}
```

## 文档值

​     在 Elasticsearch 中，Doc Values 是一种列式存储结构。Doc Values 默认对所有字段启用，除了 text 和 annotated_text 类型字段。

​    Doc Values 是在索引时创建的，当字段索引时，Elasticsearch 为了能够快速检索,会把字段的值加入倒排索引中，同时它也会存储该字段的 Doc Values。

​    Elasticsearch 中的 Doc Values 常被应用到以下场景：

- 对一个字段进行排序 
- 对一个字段进行聚合 
- 某些过滤，比如地理位置过滤 
- 某些与字段相关的脚本计算 
- 使用 docvalue_fields 返回搜索结果部分字段值

​    因为文档值被序列化到磁盘，可以依靠操作系统的帮助来快速访问。当数据集远小于节点的可用内存，操作系统会自动将所有的 Doc Values 保存在内存中，使得其读写十分高速；当其远大于可用内存，操作系统会自动把 Doc Values 加载到系统的页缓存中，从而避免了 JVM 堆内存溢出异常。



## 列式存储的压缩

   Doc Values 本质上是一个序列化的列式存储，适用于聚合、排序、脚本等操作。 这种存储方式也非常便于压缩，特别是数字类型，这样可以节约磁盘空间，并且提高访问速度。

  来看一组数字类型的 Doc Values，了解它如何压缩数据：

| 文档   | 词条 |
| ------ | ---- |
| 文档 1 | 100  |
| 文档 2 | 1000 |
| 文档 3 | 1500 |
| 文档 4 | 1200 |
| 文档 5 | 300  |
| 文档 6 | 1900 |
| 文档7  | 4200 |

​     按列布局意味着我们有一个连续的数据块：[100,1000,1500,1200,300,1900,4200]。  

​     因为我们已经知道他们都是数字，而不是像文档或行中看到的异构集合，所以我们可以使用统一的偏移来将他们紧紧排列。

​    而且，针对这样的数字有很多种压缩技巧。你会注意到这里每个数字都是 100 的倍数，Doc Values 会检测一个段里面的所有数值，并使用一个最大公约数，方便做进一步的数据压缩。

​    如果我们保存 100 作为此段的除数，我们可以对每个数字都除以 100，然后得到：[1,10,15,12,3,19,42]。

​    现在这些数字变小了，只需要很少的位就可以存储下，也减少了磁盘存放的大小。Doc Values 在压缩过程中使用如下技巧，它会按依次检测以下压缩模式:

- 如果所有的数值各不相同或缺失，设置一个标记并记录这些值 
- 如果这些值小于 256，将使用一个简单的编码表 
- 如果这些值大于 256，检测是否存在一个最大公约数 
- 如果没有存在最大公约数，从最小的数值开始，统一计算偏移量进行编码



​    你会发现这些压缩模式不是传统的通用压缩算法，比如 DEFLATE 或者 LZ4。因为列式存储的结构是严格且良好定义的，我们可以通过使用专门的模式来达到，比通用压缩算法（如 LZ4）更高的压缩效果。



## 禁用 Doc Values

​     Doc Values 默认对所有字段启用，除了 text 和 annotated_text 类型字段。也就是说所有的数字、地理坐标、日期、IP 和 keyword 类型都会默认开启。

​     Text 类型字段不能使用 Doc Values，文本经过分析流程生成很多 Token，使得Doc Values 不能高效运行。

​     因为 Doc Values 默认启用，你可以选择对你数据集里面的大多数字段，进行聚合和排序操作。如果你知道你永远也不会对某些字段进行聚合、排序或是使用脚本操作，你可以通过禁用特定字段的 Doc Values。这样不仅节省磁盘空间，也会提升索引的速度。

​     要禁用 Doc Values，在字段的映射 (mapping) 设置 doc_values: false 即可。例如，这里我们创建了一个新的索引，字段 "session_id" 禁用了 Doc Values：

```shell
PUT my_index 
{
    "mappings":{
        "properties":{
            "session_id":{
                "type":"keyword",
                "doc_values":false
            }
        }
    }
}
```

​      通过设置 doc_values: false，这个字段将不能被用于聚合、排序以及脚本操作。

​      反过来也是可以进行配置的：让一个字段可以被聚合，通过禁用倒排索引，使它不能被正常搜索，例如：



```shell
PUT my_index 
{
    "mappings":{
        "properties":{
            "customer_token":{
                "type":"keyword",
                "doc_values":true,
                "index":false
            }
        }
    }
}
```

​     通过设置 doc_values: true 和 index: false，我们得到一个只能被用于聚合/排序/脚本的字段。无可否认，这是一个非常少见的情况，但很有用。

## 存储

​     默认情况下，字段原始值会被索引用于查询，但是不会被存储。为了展示文档内容，通过一个叫 _source 的字段用于存储整个文档的原始值。

​    在字段的映射 (mapping) 设置 store: true，可以使索引单独保存这个字段。通常 情况下，如果文档本身十分庞大，而一些字段又会经常单独使用，那么这样的字段，就可以设置为单独存储，然后可以使用 stored_fields 单独检索这些字段。

​    例如，如果你的文档包含标题、时间和一个很大的正文字段，你可能只需要检索标题、时间字段，没必要从很大的 _source 原文中解析出这些字段：



```shell
#创建索引,指定常用字段 store 属性 
PUT /my-index-000001
{
    "mappings":{
        "properties":{
            "title":{
                "type":"text",
                "store":true
            },
            "date":{
                "type":"date",
                "store":true
            },
            "content":{
                "type":"text"
            }
        }
    }
}

#插入记录 
PUT /my-index-000001/_doc/1 
{
    "title":"短文本标题",
    "date":"2021-05-01",
    "content":"很长很长很长的正文字段..."
}

#查询结果返回 stored_fields 指定字段 
GET /my-index-000001/_search 
{
    "stored_fields":[
        "title",
        "date"
    ]
}
```

​    注意：stored_fields 返回结果是数组格式。如果你需要获取原始文档，可以通过_source 字段替代。



## 原文

   _source 字段包含索引时发送的原始 JSON 文档。_source 字段本身不建索引，但是存储原始文档，以便在执行查询请求时，可以将其返回。可以通过设置，禁用原文字段，或者只存储特定字段。

  _source 在 Lucene 中是映射为一个特殊的字段：

| Field   | Index | IndexType | Analyer | DocValues | Store |
| ------- | ----- | --------- | ------- | --------- | ----- |
| _source | No    | No        | No      | No        | Yes   |

   Elasticsearch 中 _source 字段的主要目的，是通过 doc_id 读取该文档的原始内容，所以只需要存储 Store 即可。

   Elasticsearch 中使用 _source 字段可以实现以下功能：

   **Update：**

   部分更新时，需要从读取文档保存在 _source 字段中的原文，然后和请求中的部分字段合并为一个完整文档。如果没有 _source，则不能完成部分字段的 Update 操作。

**Reindex：**

   可以通过 Reindex API 完成索引重建，过程中不需要从其他系统导入全量数据，而是从当前文档的 _source 中读取。如果没有 _source，则不能使用 Reindex API。

**Script：**

​    不管是 Index 还是 Search 的 Script，都可能用到存储在 Store 中的原始内容，如果禁用了 _source，则这部分功能不再可用。



**Summary：**

摘要信息也是来源于 _source 字段。



## 禁用 _source

```shell
PUT my-index-000001
{
    "mappings":{
        "_source":{
            "enabled":false
        }
    }
}
```

​    注意：禁用 _source 会导致更新、重建索引、摘要功能不可用，生产环境慎用。考虑节省存储空间，可以通过修改索引设置 index.codec 提高压缩效率。



## 包含/排除部分字段

​       包含/排除 _source 部分字段可以按以下方式设置它：

```shell
PUT logs 
{
    "mappings":{
        "_source":{
            "includes":[
                "*.count",
                "meta.*"
            ],
            "excludes":[
                "meta.description",
                "meta.other.*"
            ]
        }
    }
}

PUT logs/_doc/1
{
    "requests":{
        "count":10,
        "foo":"bar"
    },
    "meta":{
        "name":"Some metric",
        "description":"Some metric description",
        "other":{
            "foo":"one",
            "baz":"two"
        }
    }
}

GET logs/_search 
{
    "query":{
        "match":{
            "meta.other.foo":"one"
        }
    }
}
```



## 参考

###### ES系列十一、ES的index、store、_source、copy_to和all的区别

https://www.cnblogs.com/wangzhuxing/p/9527151.html

###### Elasticsearch 理解mapping中的store属性

https://www.cnblogs.com/sanduzxcvbnm/p/12157453.html

###### ElasticSearch中_source、store_fields、doc_values性能比较

https://blog.hufeifei.cn/2021/10/DB/source-stored-docvalues/index.html
