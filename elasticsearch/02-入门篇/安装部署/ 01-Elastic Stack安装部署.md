Elasticsearch （简称 ES ）的安装和部署，将会从以下几个方面进行阐 

述：

- 环境准备
- 系统级别参数配置 
- 下载、安装及启动 
- 常见问题及解决方案

## **环境准备**

### 环境选择策略

#### 操作系统

绝大部分ES集群的部署环境都是基于公有云Linux 的。选择 CentOS 7 作为基础系统有以下一些考虑：

- ES 虽然是基于 JVM 上运行的 Java 项目，但它在启动、运行时会对一些环境参数， 如虚拟内存数、文件句柄等有所要求。

- 国内的 ES 使用和部署中，以 CentOS 和 Debian 为主，存在少量的 Ubuntu 和 极少量的 Windows 的服务器。

  

​      官方团队对 64 位系统进行了稳定性测试和系统性兼容，其中除了以下一些版本以 

外都可以较好的支持：

- Debian 家族（7～10）中， Debian 7 从 5.x 版本开始不支持 。
- Ubuntu 家族（14、16、18）中，Ubuntu 14.04 从 7.x 版本开始不支持，但 Ubuntu 16，18 都支持 7.x
- Windows 家族（Windows server 2012/R2、2016、2019）中，只有 Windows  server 2019 对 ES 7.7 之前的版本兼容性有限。

   再结合通用云厂商和自建服务器的操作系统选型中，CentOS 7 得到了较好的支持 和维护，所以此处我们选择以 CentOS 7 作为首选操作系统。

####   内存、CPU

​     ES 节点启动的默认需求为 1C2G （1 核 CPU，2GB 内存） 

​    通过调整 $ES_HOME/config/jvm.options 文件中的堆栈配置，也可以让 ES 实例 在 1C1G 甚至更小资源的服务器上启动。 

**注意：**更小的可用资源意味着更差的性能和节点稳定性，甚至节点启动失败。 

​    实际生产中，更大的内存意味着更高的数据处理能力，更多的 CPU 核数可以支持 更多的内部线程，但是无可避免的，更多的资源也意味着更高的系统开销。一般情况， 内存和 CPU 的配比大致为 1:2 到 1:4 用以支持绝大部分数据存取、聚合等操作的使 用，但是实际的内存、CPU 的用量和配比还需要用真实数据模拟真实生产环境的压测结 果为准。

   一般的内存、CPU 配置策略大致为以下几种： 

- master 节点需要适当大小的内存
- coordination 节点需要较大的内存和 CPU 
- ingest 节点需要较大的 CPU 
- data 节点需要较大的内存和硬盘

​     实际生产中，需要通过压测来确定最佳的内存、CPU 配比，一般情况服务器的内存 除了系统占用的固定内存之外，会建议设置为服务器可用内存的一半。

​     除了 ES 实例之外，ES 所维护的 Lucene 也是 Java 库，也需要占用相应的内存。 此时，ES 的最大/小堆栈内存建议不超过 31G，否则会因为指针压缩的原因白白浪费内存资源，甚至可能出现数据存取更慢的问题。如果目标服务器的可用内存超过 64G 的话，可以考虑通过端口的配置部署多个 ES 实例。

​    

####   磁盘

- 没做特殊配置的话，ES 会在写入及不断的查询过程中，将数据集中存在最新修改 和召回的数据缓存在节点 /集群的各级内存（缓存）中。同时将绝大部分数据存在磁盘中的各种索引文件中，仅在内存中保留一部分索引文件的索引以加速数据的读 写。

- 不同于内存中的文件，ES 放置在磁盘中的文件的读写是随机的不是顺序的，所以 更快的随机读写速度将使 ES 提供更快的数据存取速度。

- 在在线搜索等高频存取的场景中，更建议使用固态硬盘以支持数据的高速读写。 

- 在离线的日志存储等低频读取的场景中，则可以考虑用机械硬盘来节约成本。

  

####   JDK

- ES 作为一个 Java 应用，也需要运行在与之相匹配的 JVM 上。 
- 相对于低版本的 ES，高版本的 ES 部署包会自带 JDK，所以只需要相对于低版本的 ES，高版本的 ES 部署包会自带 JDK，所以只需要保证部署 ES的系统可以支持对应的 JDK 就好。
- 目前 ES 7.10 所对应的 JDK 是 JDK11 至 15，所以部署的系统只要能支持 JDK11 以上的 JDK 版本都可以用来做部署。
- ES 的部署建议尽量使用 ES 自带的 OpenJDK，因为 Elastic 团队会在每个版本中对对应的 OpenJDK 版本进行适配和调教，贸然使用其他版本的 JDK 可能会带来不可预见的问题。



## 实际系统配置

**修改源并安装必要工具**

```shell
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \ -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.ustc.edu.cn/ centos|g' \ -i.bak \ /etc/yum.repos.d/CentOS-Base.repo && \ yum makecache && \ yum update -y && \ yum install -y epel-release && \ yum install -y curl wget htop unzip && \ yum install -y docker docker-compose
```



**开启docker服务**

```shell
systemctl start docker 
systemctl enable docker
```



## **系统级别参数配置**

###  配置策略   

  ES 作为一个复杂的系统，对于服务器资源的要求相较于一般的服务要更严格，这样也能够保证 ，ES 节点可以更好的发挥其作用。本节将从各种系统级别的参数要求及修改意义方面进行阐述。

- **线程数配置**：ES 内部会启动包括而不仅限于 query 线程池、数据写入线程池、数据 refresh 线 

  程池、segment merge 线程池等等。在启动时 ES 会要求系统中单个进程可用线程数 

  超过 65535。

- **文件具柄数**：在 Linux 里万物皆文件，线程也可以看作一种特殊的文件。在启动时 ES 会要求系统中可打开的文件句柄数超过 65535。

- **缓存交换**：在运行时，ES 会建议避免在运行过程中因为系统的缓存交换而产生的性能损耗。大部分操作系统有可能会将系统缓存中的数据交换到硬盘中。在 ES 节点部署的时候建议禁止这一交换行为。

- **内存锁定**：在运行时，ES 会占用大量的内存进行一系列的数据处理。建议开启内存锁定的配置，将它所占用的内存进行锁定。



### **配置流程**

**调整机器中每个进程可以拥有的 VMA(虚拟内存区域)的数量**

- 修改文件：/etc/sysctl.conf
- 添加/修改一行：vm.max_map_count=655360 
- 否则可能会遇到报错：max virtual memory areas vm.max_map_count [65530] 

is too low, increase to at least [262144]



**调整机器中每个进程可打开的文件句柄数量**

- 修改文件：/etc/security/limits.conf

- 添加/修改两组：（*作用于所有用户，主要用于服务器直接部署 ES；elasticsearch作用于 elasticsearch 用户，主要用于服务器 rpm 包部署）

  soft nofile 65535 => * soft nofile 655350 

  hard nofile 65535 => * hard nofile 655350 

  elasticsearch soft nofile 65535 => elasticsearch soft nofile 655350

  elasticsearch hard nofile 65535 => elasticsearch hard nofile 655350 

  否则可能遇到报错：max file descriptors [65535] for elasticsearch process 

  is too low, increase to at least [65536]



**开启内存锁定配置**

- 修改文件：/etc/security/limits.conf

- 添加/修改两组：
  - soft memlock 65535 => * soft memlock unlimited 
  - hard memlock 65535 => * hard memlock unlimited 
  - elasticsearch soft memlock 65535 => elasticsearch soft memlock unlimited 
  - elasticsearch hard memlock 65535 => elasticsearch hard memlock unlimited
  - 否则可能在开启了内存锁定时（bootstrap.memory_lock: true）遇到报错： memory locking requested for elasticsearch process but memory is not locked



**关闭内存交换区**

```shell
$. swapoff -a
```



**创建 ES 使用账号**



- ES 在启动时默认不允许使用 root 账户，所以需要预先创建 ES 自己的账户 
  -  useradd -m elasticsearch 



**然后通过命令切换到 elasticsearch 账户中进行后续操作**

```shell
su elasticsearch
```



## **安装实战**

   我们查询：https://www.elastic.co/cn/downloads/past-releases#elasticsearch



### **tar 包安装**

下载链接（后面以 ES_DOWNLOAD_URL 指代）： 

<img src="/Users/xiexinming/blog/elasticsearch/image/14.jpg" alt="14" style="zoom:50%;" />

https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.13.4-linux-x86_64.tar.gz

如果网络比较缓慢的话可以使用国内镜像:

https://mirrors.huaweicloud.com/elasticsearch/7.13.4/elasticsearch-7.13.4-linux-x86_64.tar.gz

#### 下载并解压：

- mkdir -p /usr/local/elasticsearch 

- cd /usr/local/elasticsearch 

- wget -c  https://mirrors.huaweicloud.com/elasticsearch/7.13.4/elasticsearch-7.13.4-linux-x86_64.tar.gz

- tar vxf elasticsearch-7.13.4-linux-x86_64.tar.gz 

  <img src="/Users/xiexinming/blog/elasticsearch/image/15.png" alt="15" style="zoom:50%;" />

  总用量 620

   README.asciidoc-->项目说明文档

  LICENSE.txt-->协议

  NOTICE.txt-->一些协议 的说明以及违反后果的警告

  plugins--> 插件文件夹，目前为空，自定义插件会放置在这里

  logs-->日志文件夹

  lib-->基础依赖库

  bin-->ES 内置的命令行工具，包括启动、密码生成等

  jdk-->ES 自带的 jdk

  modules-->ES 内置的各种功能模块，包括 Xpack 等

  config-->ES 的配置目录 



#### **最简启动**

- 确认自己处于 非 root 用户，否则后续启动会报错 

  <img src="/Users/xiexinming/blog/elasticsearch/image/16.png" alt="16" style="zoom:50%;" />

- cd elasticsearch-7.13.4

- ./bin/elasticsearch 

- 如果要后台启动，只需在启动命令后面加上 -d 

  - ./bin/elasticsearch -d 
  - 完整路径 ./usr/local/elasticsearch/elasticsearch-7.13.4/bin/elasticsearch-d 

在出现类似这些日志的时候，代表节点启动完成



如果上面使用elasticsearch用户还报如下错误：

<img src="/Users/xiexinming/blog/elasticsearch/image/17.png" alt="17" style="zoom:50%;" />

 我们可以使用chown命令，ch这里代表change（改变）的意思，own代表英文单词的owner(拥有者)，连在一起就是 change owner ,改变某个文件或者文件夹的拥有者; 我们使用chown授权此es所在目录给用户elasticsearch：

```shell
 chown -R elasticsearch elasticsearch-7.13.4
```



#### **ES 启动状态校验**

You Know, for Search. 

```shell
[root@host-10-10-10-4 elasticsearch-7.13.4]# curl http://localhost:9200
{
  "name" : "host-10-10-10-4",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "bIHS7CueRTOE4ijhrrarRQ",
  "version" : {
    "number" : "7.13.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "c5f60e894ca0c61cdbae4f5a686d9f08bcefc942",
    "build_date" : "2021-07-14T18:33:36.673943207Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```



#### **集群状态**

```shell
[root@host-10-10-10-4 elasticsearch-7.13.4]# curl localhost:9200/_cat/health?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1643447017 09:03:37  elasticsearch green           1         1      0   0    0    0        0             0                  -                100.0%
```



#### **节点停机**

- ps -ef | grep elasticsearch | grep -v grep | awk '{ print$2 }' | xargs kill -15 
- 没有后台启动的话，直接 ctrl + c 会输出类似以下的日志





### **Docker 安装**  

   先确保Centos机器上已经安装了docker engine并且已经启动。

#### **下载对应镜像**

```shell
docker pull elasticsearch:7.13.4
```

**如果目标机器无法上网，可以尝试通过其他机器下载并导入镜像**

- 在宿主机下载镜像 docker pull elasticsearch:7.13.4
- 把镜像导出为文件 docker save -o elasticsearch-7.13.4-image.tar docker.io/elasticsearch:7.13.4 
- 把导出的文件拷贝到目标机器 scp elasticsearch-7.13.4-image.tar root@192.168.1.100:/tmp 
- 登陆目标机器 ssh root@192.168.1.100 
- 导入目标镜像 docker load < elasticsearch-7.13.4-image.tar.



#### **镜像校验**

docker images

<img src="/Users/xiexinming/blog/elasticsearch/image/18.png" alt="18" style="zoom:100%;" />



#### **最简启动**

```shell
docker network create elastic
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.13.4
docker run -it -d --rm --name es01-test --net elastic -p 0.0.0.0:9200:9200 -p 0.0.0.0:9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.13.4
```



如果需要开启包括禁止交换区、文件句柄限制等设置： 

- 内存锁定：--ulimit memlock=-1:-1 

- 打开文件上限：--ulimit nofile=655350:655350 

- 关闭交换区：-e "bootstrap.memory_lock=true" 

- 后台运行：-d 

- 完整命令：

  ```shell
  docker run -it  -d  --rm  --ulimit memlock=-1:-1  --ulimit nofile=655350:655350 --name es01-test --net elastic -p 127.0.0.1:9200:9200 -p 127.0.0.1:9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.13.4
  ```



通过命令查看日志:docker logs -f es01-test

![19](/Users/xiexinming/blog/elasticsearch/image/19.png)



出现以下日志内容代表启动成功



#### **ES 启动状态校验**

同上



#### **节点停机**

通过命令 docker ps -as 找到对应的 docker container

```
xiexinming@bogon ~ % docker ps -as 
CONTAINER ID   IMAGE                                                  COMMAND                  CREATED       STATUS       PORTS                                                NAMES        SIZE
03eca81f7874   docker.elastic.co/kibana/kibana:7.13.4                 "/bin/tini -- /usr/l…"   2 weeks ago   Up 11 days   127.0.0.1:5601->5601/tcp                             kib01-test   36B (virtual 1.35GB)
cb71f81b72b7   docker.elastic.co/elasticsearch/elasticsearch:7.13.4   "/bin/tini -- /usr/l…"   2 weeks ago   Up 2 weeks   127.0.0.1:9200->9200/tcp, 127.0.0.1:9300->9300/tcp   es01-test    96.5MB (virtual 1.12GB)
```

关掉对应的 container docker stop es01-test

![20](/Users/xiexinming/blog/elasticsearch/image/20.png)





## **开发模式 VS 生产模式**

​    本节中的 ES 是最简安装、启动，所以是以单节点（single-node）的方式启动。单节点启动默认是开发模式，会忽略绝大部分的启动校验。在不确定生产模式的强制校验项有哪些时，建议所有的部署节点的初始化流程都按上文中的配置流程逐一进行配置。

​    生产模式启动强制校验项：



| 配置文件                  | 说明             | 配置项                | 默认值        | 期望值                   |
| ------------------------- | ---------------- | --------------------- | ------------- | ------------------------ |
| conf/jvm.options          | 最大/小内存      | -Xmx -Xms             | -Xmx1g -Xms1g | 部署节点可用内存的一半   |
| elasticsearch.yml         | 内存锁定         | bootstrap.memory_lock | NA            | True                     |
| /etc/sysctl.conf          | 虚拟内存区域     | vm.max_map_count      | 65530         | >262144                  |
| /etc/security/limits.conf | 文件可打开具柄数 | soft/hard nofile      | 65536         | 655350                   |
| /etc/security/limits.conf | 内存锁定         | soft/hard memlock     | NA            | unlimited                |
| /etc/security/limits.conf | 虚拟内存         | soft/hard as          | NA            | unlimited                |
| /etc/security/limits.conf | 最大进程数       | soft/hard nproc       | NA            | soft/hard nproc NA 10240 |
| /etc/security/limits.conf | 最大文件空间     | soft/hard fsize       | NA            | unlimited                |



## **集群的组建**

​     单个的 ES 节点可以支持普通的测试，但是对于生产的使用，特别是对数据安全性、可靠性、性能等维度有要求的使用中，应考虑使用 ES 集群支持生产的使用。本节将根据安装方式的不同，分别对集群的组建配置等进行述。

###  **集群组建流程**

​     ES 节点在启动时，会根据集群的信息以及自己的身份（配置在 $ES_HOME/config/elasticsearch.yml 里）尝试加入集群，其主要参数为。

- **cluster.name**: 集群名称，节点会在同一网段中尝试找到和自己同一个集群的其他节点组建/加入现有集群。

- **node.name**：节点自身的标记，集群内唯一。 

- **node.master、node.data、node.ingest**：节点身份，节点启动时会根据这个参数来进行注册。

- **network.host**： 节点监听 IP 地址，一般建议本机 IP。

- **http.port**：节点监听端口，默认 9200 

  - 不额外设置的话，集群节点间通信的端口会默认为 9300 
  - 也可以通过参数 transport.port 来特殊指定为其它的端口 

- **cluster.initial_master_nodes**：集群第一次初始化时的候选 master 节点列表。 

- **discovery.seed_hosts**：集群节点发现会尝试访问的节点列表。 

- ES 集群在启动/加入会根据节点的属性（master、master-eligible、data……）进 

  行选主和状态同步，（注：集群选主流程将在后续章节进行详细阐述，本章只进行 

  简要描述）

  - 集群初始化的时候（7.x），master 候选人（master）会尝试访问同一网段中同一个集群的所有节点。
  - 如果是新集群启动，配置在 cluster.initial_master_nodes 里的节点会优先成为master 候选节点。
  - 这些候选节点会发出投票及拉票请求给所有具有投票资格的节点（master、 data、voting_only…）。 
  - 在获取足够多的票数之后，master 节点当选，否则 master 候选节点会在等待一段时间之后重新发起投票。请注意，为了防止集群脑裂的发生，这里这里建议在 cluster.initial_master_nodes 参数中设置奇数个（只需 1～3 个节点，不一定需要所有 master 候选节点）即可。 
  - master 会通过两段提交的方式将集群的信息、组织架构、模板等信息发布给集群中的所有节点。
  - 当所有节点都成功同步集群状态之后，集群启动宣告完成。

### **tar 和 rpm 包安装方式**

   登陆每个 ES 节点，并修改配置文件并和其他节点组成集群。

我们搭建部署了3台es节点：其对应的ip地址为:

- tar 包安装的配置文件 vi /usr/local/elasticsearch/config/elasticsearch.yml 

- rpm 包安装的配置文件 vi /etc/elasticsearch/elasticsearch.yml 

  

这里的 network.host 也可以配置为 _site_方便在节点批量初始化时进行配置。

节点1：192.168.2.11

```shell
#节点1:node-11->192.168.2.11
cluster.name: es-cluster 
node.name: node-11 
network.host: 192.168.2.11 
http.port: 9200 
discovery.seed_hosts: ["node-11","node-12","node-13"] 
cluster.initial_master_nodes: ["node-11"]

```

  节点2：192.168.2.12

```shell
# IP: 192.168.2.12 
cluster.name: es-cluster 
node.name: node-12 
network.host: 192.168.2.12 
http.port: 9200 
discovery.seed_hosts: ["node-11","node-12","node-13"] 
cluster.initial_master_nodes: ["node-11"]
```

   节点3：192.168.2.13

```shell
# IP: 192.168.2.13 
cluster.name: es-cluster 
node.name: node-13 
network.host: 192.168.2.13
http.port: 9200 
discovery.seed_hosts: ["node-11","node-12","node-13"] 
cluster.initial_master_nodes: ["node-11"]
```

   **节点启动**

systemctl start elasticsearch

### **Docker 启动的配置方式**

​    纯靠 docker run 命令方式启动 ES 集群会比较麻烦，建议通过 docker-compose方式启动。  

#### 示例 docker-compose.yml 文件:

```shell
version: "2.2" 
networks: 
  bigdata: 
    driver: bridge 
volumes: 
  es-data-11: 
    driver: local 
  es-data-12: 
    driver: local 
  es-data-13: 
    driver: local 
services: 
    es-node-11:
      image: elasticsearch:7.13.4 
      restart: always 
      container_name: node-11
      environment: 
       - node.name=node-11 
       - cluster.name=docker-cluster 
       - cluster.initial_master_nodes=node-11 
       - discovery.seed_hosts=node-11,node-12,node-13 
       - bootstrap.memory_lock=true 
       - "ES_JAVA_OPTS=-Xms512m -Xmx512m" 
     ulimits: 
       memlock: 
         soft: -1 
         hard: -1 
       volumes:
         - es-data-11:/usr/share/elasticsearch/data 
       ports: 
         - 9200:9200 
         - 9300:9300 
       networks: 
         - elastic
    es-node-12:
      image: elasticsearch:7.13.4 
      restart: always 
      container_name: node-12
      environment: 
       - node.name=node-12 
       - cluster.name=docker-cluster 
       - cluster.initial_master_nodes=node-12
       - discovery.seed_hosts=node-11,node-12,node-13  
       - bootstrap.memory_lock=true 
       - "ES_JAVA_OPTS=-Xms512m -Xmx512m" 
     ulimits: 
       memlock: 
         soft: -1 
         hard: -1 
       volumes:
         - es-data-12:/usr/share/elasticsearch/data 
       ports: 
         - 9200:9200 
         - 9300:9300 
       networks: 
         - elastic
    es-node-13:
      image: elasticsearch:7.13.4 
      restart: always 
      container_name: node-13
      environment: 
       - node.name=node-13 
       - cluster.name=docker-cluster 
       - cluster.initial_master_nodes=node-13
       - discovery.seed_hosts=node-11,node-12,node-13 
       - bootstrap.memory_lock=true 
       - "ES_JAVA_OPTS=-Xms512m -Xmx512m" 
     ulimits: 
       memlock: 
         soft: -1 
         hard: -1 
       volumes:
         - es-data-13:/usr/share/elasticsearch/data 
       ports: 
         - 9200:9200 
         - 9300:9300 
       networks: 
         - elastic

```

#### 单个docker操作

​      或者也可以在宿主机维护每个节点自己的 elasticsearch.yml 文件，并通过 -v $PATH/to/elasticsearch.yml:/usr/local/elasticsearch/config/elasticsearch.yml 的方式把这些配置文件映射到 docker container 里面进行使用。

​     所以如果需要整个集群里所有的节点都能监听/支持访问的话，需要把他们的 9200/9300 端口映射成宿主机里不同的端口，或者在 docker 环境中启动一个类似 NGINX的网关来代理所有的节点。

```shell
docker run -it  -d  --rm  --ulimit memlock=-1:-1  --ulimit nofile=655350:655350 --name es01-test --net elastic -p 0.0.0.0:9200:9200 -p 0.0.0.0:9300:9300 -v /home/vagrant/es.yml:/usr/share/elasticsearch/config/elasticsearch.yml elasticsearch:7.13.4
```

​                    

实时查看docker容器名为es01-test的日志

```shell
$ docker logs -f -t --tail -f es01-test
```

![21](/Users/xiexinming/blog/elasticsearch/image/21.png)

#### **安装elasticsearch head插件监控管理**

 我们在主节点:node-11所在机器安装head插件

```shell
docker pull mobz/elasticsearch-head:5
docker run -d --name es-head -p 0.0.0.0:9100:9100 docker.io/mobz/elasticsearch-head:5
```



打开链接 http://192.168.2.11:9100/ 查看节点状态，多个节点都显示在集群列表下，则集群安装成功

我们发现不能访问es的地址：我们打开浏览器发现：

<img src="/Users/xiexinming/blog/elasticsearch/image/22.png" alt="22" style="zoom:50%;" />

我们需要重新设置es的配置，然后添加属性:  

```shell
# head插件设置
http.cors.enabled: true   #允许跨站访问
http.cors.allow-origin: "*"    #允许跨站来源
```



<img src="/Users/xiexinming/blog/elasticsearch/image/23.png" alt="23" style="zoom:50%;" />



然后我们访问：http://192.168.2.11:9100/

![24](/Users/xiexinming/blog/elasticsearch/image/24.png)





参考：

https://www.jianshu.com/p/ced0ba26bd8d

https://www.cnblogs.com/yidiandhappy/p/7714489.html

