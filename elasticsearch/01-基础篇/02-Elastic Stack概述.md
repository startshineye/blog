

## Elastic Stack 是什么？  

   Elastic Stack 是一系列由 Elastic 公司开发的产品组件，能够安全可靠地获取任何 来源、任何格式的数据，然后实时地对数据进行搜索、分析和可视化。Elastic Stack 旧称 ELK Stack，主要有 Elasticsearch，Logstash，Kibana，Beats 四种组件组成。 

​    <img src="/Users/xiexinming/blog/elasticsearch/image/1.jpg" alt="1" style="zoom:25%;" />



## 特点

- #### 配置简单，开箱即用

  上手简单，只需要修改几行配置，就能快速搭建起一套 Elastic Stack 平台。

- #### 多种数据源支持

  Beats 组件支持多种数据类型，能够快速满足日志，指标，网络数据等多种数据源 接入的需求。

- ### 性能优异

  无论是数据写入还是实时检索，性能都相当优异。

- #### 集群扩展性强

  Elasticsearch 和 Kibana 均可以进行灵活扩展，能够有效提高集群性能和高可用 性。

- #### 多维度分析

  得益于 Elasticsearch 强大的搜索能力和 Kibana 优秀的可视化功能，能够从多个 维度，对数据进行聚合分析并且展示。

  

  ## Elastic Stack应用场景

  Elastic Stack 除了采集提供采集各类日志的能力外，它还提供了许多开箱即用的场 景：企业搜索，可观测性，安全解决方案等。  

  <img src="/Users/xiexinming/blog/elasticsearch/image/2.png" alt="2" style="zoom:30%;" />



#### 企业搜索

​    Elastic 企业搜索由 Elasticsearch 提供支持，性能优异，同时其采用的相关性模型 已针对实际的自然搜索进行优化且得到了实践验证，因此能够快速投入应用。同时它又 提供了灵活的定制选项，可以让客户快速根据自身需要打造精致又自然的搜索体验。

#### 可观测性

   Elastic Stack 将你的可观测性数据统一到一个强大的数据存储中，便于你实时搜索 并应用交互式分析。凭借日志、指标和 APM 追踪之间的直观导航，便能依赖 Machine Learning 暴露异常数值，并对你系统中发生的所有事件采取对策。同时提供良好的可视 化体验，能够以最直观的方式展示你的数据，完美掌握系统运行状态。



## Elasticsearch 在 Elastic Stack 中的位置及能力

   Elasticsearch 提供数据的存储及检索能力，并且它是最 核心的组件。外部数据采集到 Elasticsearch 后，用户便可针对他们的数据运行复杂的 查询，并使用聚合来检索自身数据的复杂汇总。



### Elasticsearch Index

- **是什么？**是相互关联的文档集合：Elasticsearch 以 JSON 文档 的形式存储数据。每个文档都会在一组键（字段或属性的名称）和它们对应的值（字符 串、数字、布尔值、日期、数组自值、地理位置或其他类型的数据）之间建立联系
- **底层数据结构**：为倒排索引的数据结构，这一结构的设计可以允许十分快速地进行全文本搜索。倒排索引会列出在所有文档中出现的每个特有词汇，并且可以找到包含每个词汇的全部文档。

### Elasticsearch 能力

- **Elasticsearch 很快**

  Elasticsearch 同时还是一个近实时的搜索平台，这意味着从文档索引操作到 文档变为可搜索状态之间的延时很短，一般只有一秒（这个可以同过配置进行调整）。 因此，Elasticsearch 非常适用于对时间有严苛要求的用例，例如安全分析和基础设施监测。

- **Elasticsearch 具有分布式的本质特征**

  Elasticsearch 中存储的文档分布在不同的容器中，这些容器称为分片，可以进行复 制以提供数据冗余副本，以防发生硬件故障。Elasticsearch 的分布式特性使得它可以扩 展至数百台（甚至数千台）服务器，并处理 PB 量级的数据。

- **Elasticsearch 优异的相关性检索能力**

  Elasticsearch 底层采用倒排索引的数据结构，检索过程中采用优异的相关性计算算 法，因此 Elasticsearch 具有优异的相关性检索能力，它能基于多个维度多搜索结果进 行评分，因此它可以与业务需求进行结合，达到满足用户需求的搜索结果。

- **Elasticsearch 包含一系列广泛的功能**

  除了速度、可扩展性和弹性等优势以外，Elasticsearch 还有大量强大的内置功能（例 如数据汇总和索引生命周期管理），可以方便用户更加高效地存储和搜索数据。



## Logstash 在 Elastic Stack 中的位置及能力

#### Logstash 在 Elastic Stack 中的作用

​    Logstash 是一个开源的服务器端数据处理管道，允许你在将数据索引到 Elasticsearch 之前同时从多个来源采集数据，并对数据进行丰富和转换。

#### Logstash 能力

- **采集各种样式、大小和来源的数据**  

  Logstash 支持多 种输入选择，可以同时从众多常用来源捕捉事件。能够以连续的流式传输方式，轻松地 从你的日志、指标、Web 应用、数据存储以及各种 AWS 服务采集数据。

- **实时解析和转换数据**

  数据从源传输到存储库的过程中，Logstash 过滤器能够解析各个事件，识别已命名 的字段以构建结构，并将它们转换成通用格式，以便进行更强大的分析和实现商业价值。

  Logstash 能够动态地转换和解析数据，不受格式或复杂度的影响，比如：  利用 Grok 从非结构化数据中派生出结构  从 IP 地址破译出地理坐标  将 PII （Personal Identifiable Information）数据匿名化，完全排除敏感字段  简化整体处理，不受数据源、格式或架构的影响。

- **Logstash 扩展性**

  Logstash 采用可插拔框架，拥有 200 多个插件。  



## Kibana 在 Elastic Stack 中的位置及能力

#### Kibana 在 Elastic Stack 中的作用

​    Kibana 是一款适用于 Elasticsearch 的数据可视化和管理工具，可以提供实时的直 方图、线形图、饼状图和表格等等。可将 Kibana 作为用户界面来监测和管理 Elastic Stack 集群并确保集群安全性，还可将其作为基于 Elastic Stack 所开发内置解 决方案的汇集中心。

#### Kibana 能力

Kibana 与 Elasticsearch 和更广意义上的 Elastic Stack 紧密集成，这一点使其成 为支持下列场景的理想之选，搜索、查看并可视化 Elasticsearch 中所索引的数据，并通过创建柱状图、饼状图、 表格、直方图和地图对数据进行分析。仪表板视图能将这些可视化元素集中到一起，然 后通过浏览器加以分享，以提供有关海量数据的实时分析视图，为下列用例提供支持： 

- 日志处理和分析 

- 基础设施指标和容器监测 

- 应用程序性能监测 (APM) 

- 地理空间数据分析和可视化 

- 安全分析 

- 业务分析 

- 机器学习 

  借助网络界面来监测和管理 Elastic Stack 实例并确保实例的安全。



## Beats在Elastic Stack中的位置及能力

#### Beats 在 Elastic Stack 中的作用  

Beats 在 Elastic stack 中是数据的采集器，Beats 可以从你的各种环境中收集日 志，指标，安全数据或者网路数据，然后通过来自主机、诸如 Docker 和 Kubernetes 等容器平台以及云服务提供商的必要元数据对这些内容进行记录，然后再传输到 ElasticStack 中。

#### Beats 诞生的原因

ELK 在最初仅包含 Elasticsearch，Kibana，Logstash。在旧有的日志采集系统中， 数据管道包含 3 个主要阶段，数据采集，数据处理和存储，其中的前两个阶段均由 Logstash 进行承担。然后由于 Logstash 的设计导致的内在问题，常常发生性能问题， 尤其是在有复杂的管道处理流程中。因此，转移 Logstash 的想法也应用而生，因此将 数据提取任务抽离之后，就诞生了 Beats。

#### Beats 优点

- 即插即用

​       Beats 能够简化从关键数据源（例如云平台、容器和系统，以及网络技术）采集、 解析和可视化信息的过程。只需一行命令，就能完成数据的采集。 

- 扩展项强 

​       Beats 提供了大量不同类型的采集器，同时提供了自定义协议所需的构建基石，方 便扩展，同时 Beats 社区在不断状态，未来将会诞生满足更多场景的 Beats。 

- 轻量易部署 

​      相较于 Logstash，Beats 体积更小，性能更高，能够快速接入不同的数据源进行采 集。



#### Beats 系列

​    Beats 提供了多种类型的采集器，方便你即插即用，搞定大多数数据类型的采集。

- Filebeats ：

采集日志文件 Filebeat 随附可观测性和安全数据源模块，这些模块简化了常见格式的日志的收集、 解析和可视化过程，只需一条命令即可。之所以能实现这一点，是因为它将自动默认路 径（因操作系统而异）与 Elasticsearch 采集节点管道的定义和 Kibana 仪表板组合在 一起。不仅如此，Filebeat 的一些模块还随附了预配置的 Machine Learning 作业。 

- Metricbeat ：

采集指标 将 Metricbeat 部署到你的所有 Linux、Windows 和 Mac 主机，并将它连接到 Elasticsearch 就大功告成了：你可以获取系统级的 CPU 使用率、内存、文件系统、磁 盘 IO 和网络 IO 统计数据，还可针对系统上的每个进程获得与 top 命令类似的统计数据。 

- Packetbeat：

 采集网络数据 HTTP 等网络协议能够让你密切监测应用程序延迟和错误、响应时间、SLA 性能、 用户访问模式和趋势等等。 而 Packetbeat 则让你能够访问这些数据，了解流量的网络 传输状态。这款工具完全采用被动模式，毫无延迟开销，并且不会妨碍你的基础架构。

-  Winlogbeat ：

采集 Windows 事件日志 用于密切监控基于 Windows 的基础设施上发生的事件。使用 Winlogbeat，将 Windows 事件日志流式传输至 Elasticsearch 和 Logstash。



- Auditbeat ：

采集审计数据 你可以使用既有审计规则来轻而易举地收集数据，而无需重写规则。是谁在什么时 间做了什么事情？Auditbeat 会记住所有这些原始的系统调用数据，以及相关联的路径， 方便你了解所需的上下文信息。 

- Heartbeat ：

采集运行时间监控 无论你要测试同一台主机上的服务，还是要测试开放网络上的服务，Heartbeat 都能轻松生成运行时间数据和响应时间数据。 

- Functionbeat ：

无需服务器的采集器 你能够通过无服务器架构部署代码，省去了启动和管理额外的底层软件和硬件的麻 烦。通过 Functionbeat，你能够同样简单地监测云端基础架构。

-  Journalbeat ：

 journald 日志采集器。





