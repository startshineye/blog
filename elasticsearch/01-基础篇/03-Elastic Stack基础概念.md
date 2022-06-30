



![3](/Users/xiexinming/blog/elasticsearch/image/3.png)



#### Cluster(集群)

Elasticsearch 集群由一个或多个节点组成，可通过其集群名称进行标识。在默认的情况下，如我们的 Elasticsearch已经开始运行，那么它会自动生成一个叫做 “Elasticsearch” 的集群。当然我们可以在 config/elasticsearch.yml 里定制我们的集群的名字：

```
[root@cb71f81b72b7 config]# cat elasticsearch.yml 
cluster.name: "docker-cluster"
network.host: 0.0.0.0
[root@cb71f81b72b7 config]# 
```

一个 Elasticsearch 的集群就像是下面的一个布局：

![4](/Users/xiexinming/blog/elasticsearch/image/4.png)

​       带有 NginX 代理及 Balancer 的架构图是这样的：

​       <img src="/Users/xiexinming/blog/elasticsearch/image/5.png" alt="5" style="zoom:30%;" />



​       我们可以通过：

```shell
GET _cluster/state
```

​    来获取整个 cluster 的状态。这个状态只能被 master node 所改变。上面的接口 返回的结果是：

```
{
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "Lmv7APZ4QemXB88AGZZfcA",
  "version" : 163,
  "state_uuid" : "7nh2_aWaT9aJHiVaOo5UjQ",
  "master_node" : "bhsEXqpaT7K2PEmXSYlbsg",
  "blocks" : { },
  "nodes" : {...},
  "metadata" : {...},
  "routing_table" : {...},
  "routing_nodes" : {...}
}
```



#### Node(节点)

   节点就是单个 Elasticsearch 实例。在大多数环境中，每个节点都在单独的盒子或虚拟机上运行。一个集群由一个或多 个 node 组成。在测试的环境中，我可以把多个 node 运行在一个 server 上。在实际 的部署中，大多数情况还是需要一个 server 上运行一个 node。



  **节点分类**：

- master-eligible：可以作为主node。一旦成为主node，它可以管理整个 cluster 的设置及变化：创建，更新，删除 index；添加或删除 node；为 node 分配 shard 
-  data：数据 node 
-  ingest: 数据接入（比如 pipepline) 
-  machine learning (Gold/Platinum License) 

   

​    一般来说，一个 node 可以具有上面的一种或几种功能。我们可以在命令行或者 Elasticsearch 的配置文件（Elasticsearch.yml）来定义：



| Node类型         | 配置参数    | 默认值                  |
| ---------------- | ----------- | ----------------------- |
| master-eligible  | Node.master | True                    |
| data             | Node.data   | True                    |
| ingest           | Node.ingest | True                    |
| machine learning | Node.ml     | true (除了 OSS 发布版） |

​    你也可以让一个 node 做专有的功能及角色。如果上面 node 配置参数没有任何 配置，那么我们可以认为这个 node 是作为一个 coordination node。在这种情况下。

​    它可以接受外部的请求，并转发到相应的节点来处理。针对 master node，有时我们需 要设置 cluster.remote.connect: false。

​    在实际的使用中，我们可以把请求发送给 data 节点，而不能发送给 master 节点。我们可以通过对 config/elasticsearch.yml 文件中配置来定义一个 node 在集群 中的角色：

<img src="/Users/xiexinming/blog/elasticsearch/image/12.png" alt="12" style="zoom:50%;" />

  

在有些情况中，我们可以通过设置 node.voting_only 为 true 从而使得一个 node 在 node.master 为真的情况下，只作为参加 voting 的功能，而不当选为 master node。这种情况为了避免脑裂情况发生。它通常可以使用一个 CPU 性能较低的 node 来担当。



   我们可以使用如下的一个命令来获取当前可以进行 vote 的所有 master-eligible 节点：

```shell
GET /_cluster/state?filter_path=metadata.cluster_coordination.last_committed_config
```

得类似如下列表的结果：

```shell
{
  "metadata" : {
    "cluster_coordination" : {
      "last_committed_config" : [
        "bhsEXqpaT7K2PEmXSYlbsg",
        "chsEXqpaT7K2PEmXSYlbsg",
        "dhsEXqpaT7K2PEmXSYlbsg"
      ]
    }
  }
}
```



在整个 Elastic 的架构中，Data Node 和 Cluster 的关系表述如下：

![13](/Users/xiexinming/blog/elasticsearch/image/13.png)





#### Document(文档)  

​    Elasticsearch 是面向文档的，这意味着你索引或搜索的最小数据单元是文档。

   文档在 Elasticsearch 中有一些重要的属性：

- 独立的：

  文档包含字段(名称)及其值。

- 可以是分层的：

  可以将其视为文档中的文档，它还可以包含其他字段和值。

- 结构灵活

  文档不依赖于预定义的架构。



​     Document 相比较于关系数据库，它相应于其中每个 record。



#### Type(类型)

​    类型是文档的逻辑容器，类似于表是行的容器。

​    你将具有不同结构（模式）的文档放在不同类型中。 例如，你可以使用一种类型来 定义聚合组，并在人们聚集时为事件定义另一种类型。

​    在 Elasticsearch 中，我们开始可以不定义一个 mapping，而直接写入到我们指定 的 index 中。这个 index 的 mapping 是动态生成的 （当然我们也可以禁止这种行 为）。其中的数据项的每一个数据类型是动态识别的。比如时间，字符串等，虽然有些 数据类型，还是需要我们手动调整，比如 geo_point 等地理位置数据。



#### Index(索引)

   在 Elasticsearch 中，索引是文档的集合。  

<img src="/Users/xiexinming/blog/elasticsearch/image/6.png" alt="6" style="zoom:50%;" />

   每个 Index 由一个或多个 documents 组成，并且这些 document 可以分布于不 同的 shard 之中。

 <img src="/Users/xiexinming/blog/elasticsearch/image/7.png" alt="7" style="zoom:30%;" />

  

​     很多人认为 index 类似于关系数据库中的 database。这中说法是有些道理，但是 并不完全相同。其中很重要的一个原因是，在 Elasticsearch 中的文档可以有 object 及 nested 结构。一个 index 是一个逻辑命名空间，它映射到一个或多个主分片，并且可 以具有零个或多个副本分片。

​    每当一个文档进来后，根据文档的 id 会自动进行 hash 计算，并存放于计算出来 的 shard 实例中，这样的结果可以使得所有的 shard 都比较有均衡的存储，而不至于 有的 shard 很忙。

```shell
shard_num = hash(_routing) % num_primary_shards
```

   上面的公式我们也可以看出来，我们的 shard 数目是不可以动态修改的，否则之 后也找不到相应的 shard 号码了。必须指出的是，replica 的数目是可以动态修改的。



#### Shard

​     Elasticsearch 是一个分布式搜索引擎，因此索引通常会拆分为分布在多个节 点上的称为分片的元素。 Elasticsearch 自动管理这些分片的排列。 它还根据需要重新 平衡分片，因此用户无需担心细节。

​    由于分片的作用，一个索引可以存储单个节点硬件限制的大量数据。比如一个具备20亿，大小为2TB的磁盘空间，而任一节点都没有这样大的磁盘空间；或者单个节点处理 搜索请求，响应太慢。为了解决这个问题，Elasticsearch 提供了将索引划分成多份的能力，这些份就叫做 分片（shard）。当你创建一个索引的时候，你可以指定你想要的分片 (shard) 的数量。 每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群 中的任何节点上。

   **分片作用：**

- 允许你水平分割/扩展你的内容容量。 
- 允许你在分片（潜在地，位于多个节点上）之上进行分布式的、并行的操作，进而 提高性能/吞吐量。

   **分片种类：**

-  **Primary shard**

  每个文档都存储在一个 Primary shard。 索引文档时，它首先在 Primary shard 上编制索引，然后在此分片的所有副本上（replica）编制索引。索 引可以包含一个或多个主分片。 此数字确定索引相对于索引数据大小的可伸缩性。 创建索引后，无法更改索引中的主分片数。

- **Replica shard**

  每个主分片可以具有零个或多个副本。 副本是主分片的副本，有两 个目的：

  **增加故障转移**：如果主分片故障，可以将副本分片提升为主分片。 

  **提高性能**：get 和 search 请求可以由主 shard 或副本 shard 处理。



默认情况下，每个主分片都有一个副本，但可以在现有索引上动态更改副本数。 永远不会在与其主分片相同的节点上启动副本分片。

​      如一个索引:index 有 5 个 shard 及 1 个 replica的情况如下：

<img src="/Users/xiexinming/blog/elasticsearch/image/8.png" alt="8" style="zoom:50%;" />

​    这些 Shard 分布于不同的物理机器上：

![9](/Users/xiexinming/blog/elasticsearch/image/9.png)



   可以为每个 Index 设置相应的 Shard 数值：

```shell l
curl -XPUT http://localhost:9200/wechat?pretty -H 'Content-Type: application/json' - d ' { "settings" : { "index.number_of_shards" : 2, "index.number_of_replicas" : 1
}
}
```

   上面的 REST 接口中，我们为 wechat 这个 index 设置了 2 个 shards，并且有一个 replica。一旦设置好 primary shard 的数量，我们就不可以修改 了。这是因为 Elasticsearch 会依据每个 document 的 id 及 primary shard的数量来把相应的document 分配到相应的shard中。如果这个数量以后修改的话，那么每 次搜索的时候，可能会找不到相应的 shard。

   我们可以通过如下的接口来查看我们的 index 中的设置：

```shell
curl -XGET http://localhost:9200/wechat/_settings?pretty
```



#### Replica(副本)

​     默认情况下，Elasticsearch 为每个索引创建一个主分片和一个副本。这意味着每个 索引将包含一个主分片，每个分片将具有一个副本。

​     分配多个分片和副本是分布式搜索功能设计的本质，提供高可用性和快速访问索引中的文档。主分片和副本分片之间的主要区别在于，只有主分片可以接受索引请求。副本和主分片都可以提供查询请求。

​    **注意**：number_of_shards 值与索引有关，而与整个集群无关。此值指定每个索引的分片数（不是集群中的主分片总数）。



​    通过如下的接口来获得一个index的健康情况：

```shell
 http://localhost:9200/_cat/indices/twitter
```

![10](/Users/xiexinming/blog/elasticsearch/image/10.jpg)



更进一步的查询，我们可以看出：

<img src="/Users/xiexinming/blog/elasticsearch/image/11.png" alt="11" style="zoom:50%;" />

  

   如果一个 index 显示的是红色，表面这个 index 至少有一个 primary shard 没有 被正确分配，并且有的 shard 及其相应的 replica 已经不能正常访问。 如果是绿色， 表明 index 的每一个 shard 都有备份 （replica），并且其备份也成功复制在相应的 replica shard 之中。如果其中的一个 node 坏了，相应的另外一个 node 的 replica 将起作用，从而不会造成数据的丢失。



#### shard 健康

- 红色：集群中未分配至少一个主分片 。
- 黄色：已分配所有主副本，但未分配至少一个副本 。
-  绿色：分配所有分片。 





