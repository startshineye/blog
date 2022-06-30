 本节主要讲解kibana部署,



## **环境准备**

​    Kibana 是一个基于 Nodejs 构建出来的前端项目，它本身不包含数据存储功能， 所以需要配合一个Elasticsearch 节点/集群一起进行使用。本节将从系统环境的选择，必须的基础应用的安装等方面进行阐述。

   

### **环境选择策略**

1. **操作系统** 

​    由于 Kibana 不能独立存在，需要绑定一个 Elasticsearch 节点/集群，所以本文主要会以一个 CentOS 7 系统来承载它配套的 Elasticsearch 节点。我们也将介绍其它常用操作系统的安装。 

   Kibana 可以支持的系统和 Elasticsearch 类似，可以大致认为支持 Elasticsearch的系统都可以承载 Kibana。

2. **内存、CPU**

   ​    Kibana 是一个前端系统，绑定的 Elasticsearch 可以认为是它用来存取数据用的数据库，所以不需要特别高的配置。

      本文将以一个最小能够顺畅运行 Elasticsearch 节点的配置进行描述（1C2G）

   

### **实际系统配置**

​    Kibana 的系统安装、配置和上一节 Elasticsearch 的安装一致，修改源并安装必要工具：

```shell
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \ -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.ustc.edu.cn/ centos|g' \ -i.bak \ /etc/yum.repos.d/CentOS-Base.repo && \ yum makecache && \ yum update -y && \ yum install -y epel-release && \ yum install -y curl wget htop unzip && \ yum install -y docker docker-compose
```

​    开启 docker 服务：

```shell
systemctl start docker
systemctl enable docker
```



## **下载、安装及启动**

### **tar 包安装**

   国内镜像下载链接（后面以 KIBANA_DOWNLOAD_URL 指代）：

```shell
https://repo.huaweicloud.com/kibana/7.13.4/kibana-7.13.4-linux-x86_64.tar.gz
```

#### **下载并解压：** 

- 创建文件：mkdir -p /usr/local/kibana 

- 切换目录：cd /usr/local/kibana 

- 下载文件：wget -c https://repo.huaweicloud.com/kibana/7.13.4/kibana-7.13.4-linux-x86_64.tar.gz

- 解压：tar vxf kibana-7.13.4-linux-x86_64.tar.gz

  ```shell
  drwxr-xr-x   3 root root    4096 7月  15 2021 x-pack
  drwxr-xr-x  10 root root    4096 7月  15 2021 src
  -rw-r--r--   1 root root    3968 7月  15 2021 README.txt--->项目说明文档
  drwxr-xr-x   2 root root    4096 7月  15 2021 plugins-->插件文件夹， 目前为空，自定义插件会放置在这里
  -rw-r--r--   1 root root     740 7月  15 2021 package.json-->项目打包文件
  -rw-r--r--   1 root root 1476895 7月  15 2021 NOTICE.txt-->一些协议 的说明以及违反后果的警告
  drwxr-xr-x 827 root root   32768 7月  15 2021 node_modules
  -rw-r--r--   1 root root    3860 7月  15 2021 LICENSE.txt-->协议
  drwxr-xr-x   2 root root    4096 7月  15 2021 data-->Kibana 和它的 插件写本地文件的文件夹
  drwxr-xr-x   2 root root    4096 7月  15 2021 config-->配置文件目录
  drwxr-xr-x   6 root root    4096 7月  15 2021 node
  drwxr-xr-x   2 root root    4096 7月  15 2021 bin--->Kibana 内置命令行工具
  ```

​       修改配置文件 ${KIBANA_HOME}/config/kibana.yml

- 添加 Elasticsearch 访问地址：elasticsearch.hosts: ["http://localhost:9200"] 

- 服务启动：

  通过命令./bin/kibana 启动

  kibana 不像 ES 有直接的后台运行参数，只能通过 nohup 配合 & 的方式后台运行

  完整命令：

  ```shell
  nohup ./bin/kibana > kibana.log 2>&1 &
  ```

- 在浏览器里通过地址 http://${node_ip}:5601 进行访问



#### 服务暂停

- Kibana 的进程是一个 nodejs 服务，所以不能像 Elasticsearch 一样通过 ps -ef | grep kibana 的方式获取进程 id 
- 只能通过命令 netstat，通过查找监听端口的方式来找到 kibana 对应的 pid 并进行kill 操作 

- 完整命令：

  ```shell
  netstat -anp | grep 5601 | awk '{ print $7 }' | cut -d '/' -f 1 | xargs kill -15
  ```



### **Docker/docker-compose 安装**

**下载对应镜像：**

```shell
docker pull kibana:7.13.4
```

(可选)如果目标机器无法上网，可以尝试通过其他机器下载并导入镜像：

- 在宿主机下载镜像 docker pull kibana:7.13.4
- 把镜像导出为文件 docker save -o kibana-7.13.4-image.tar docker.io/kibana:7.13.4
- 把导出的文件拷贝到目标机器 scp kibana-7.13.4-image.tar root@192.168.10.221: /tmp 
- 登陆目标机器 ssh root@192.168.10.221 
- 导入目标镜像 docker load < kibana-7.13.4-image.tar



**镜像验证：**

```shell
 docker images
```



**服务启动：**

   不太建议直接命令行启动，因为需要和 Elasticsearch 节点配置共通网络之类的事情。

   这里主要介绍通过 docker-compose 的方式进行管理。

   修改配置文件 vi docker-compose.yml。

```shell
# 声明 docker-compose 版本，Mac 等环境可以使用 3，但是在一些 Linux 环境中只支持到 2 
version: "2.2" 
# 声明节点使用的网络空间 
networks: 
   bigdata: 
     driver: bridge 
#声明 Kibana 节点 
services: 
   kibana: 
   # kibana 版本要和 ES 相匹配，否则会报错甚至无法正常启动 
     image: kibana:7.13.4 
     container_name: kibana 
     environment: 
        # 如果 ES 节点和当前 kibana 节点在同一个 docker-compose 环境中 
        # 可以直接写对应的 ES container_name，否则需要填完整的 URL 
        ELASTICSEARCH_HOSTS: http://es01:9200 
     depends_on: - es01 
     ports: - 5601:5601 
     networks: - bigdata
```



或者我们单独启动如下：

```shell
docker run -d --name kibana  -p 0.0.0.0:5601:5601 --restart=always kibana:7.13.4
```

然后进入容器修改其es的连接地址:

<img src="/Users/xiexinming/blog/elasticsearch/image/25.png" alt="25" style="zoom:50%;" />

然后重启服务:

```shell
docker restart kibana 
```



**服务访问**

​      在浏览器里通过地址 http://192.168.2.11:5601 进行访问，刚开始可能访问比较慢，需要等待一会，然后才会出现下面的效果。

​    然后进入kibana控制台:执行:

```shell
GET /
```

得到下面的结果:

```shell
{
  "name" : "node-11",
  "cluster_name" : "es-cluster",
  "cluster_uuid" : "q2_ZSSScSYy08nV9psoa9w",
  "version" : {
    "number" : "7.13.4",
    "build_flavor" : "default",
    "build_type" : "docker",
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



![26](/Users/xiexinming/blog/elasticsearch/image/26.png)



## **安全设置开启**

​    Kibana 作为与 Elasticsearch 紧密相关的应用，在 Elasticsearch 开启了安全性认证的时候也需要相应的开启安全性认证。步骤如下:



- 在 Elasticsearch 集群中开启安全性设置 
- 通过 bin/elasticsearch-setup-passwords 命令生成所需要账户的密码 
- 修改配置文件 $KIBANA_HOME/config/kibana.yml 将 kibana 账号（7.x 之后可能会是 kibana_system 账号）对应的密码配置进相关参数中 
- elasticsearch.username: "kibana_system" 
- elasticsearch.password: "password" 
- 在配置好安全设置之后初次登陆时需要以管理员（elastic）账号进行登陆，并进行后续的配置和操作



**各环境的配置方式参考**

- rpm 包安装：/etc/kibana/kibana.yml 

- tar 包安装：$KIBANA_HOME/config/kibana.yml

- docker 安装：

  - 通过 -v 参数将本地配置文件映射到 docker 节点里：docker run -v $KIBANA_CONFIG_PATH/kibana.yml:/usr/share/kibana/config/kibana.yml docker.io/kibana:7.134

  - 修改 docker-compose.yml 文件里的 environment 配置

    ```shell
    ...
      environment: 
      # 注意这几行 
      ELASTICSEARCH_HOSTS: http://es01:9200 
      ELASTICSEARCH_USERNAME: kibana 
      ELASTICSEARCH_PASSWORD: kibana-password
    ...
    ```

    

MacOS： 

- tar 包安装：同上 
- brew 安装：brew info kibana-full 中的配置地址，本例为：/usr/local/etc/kibana/config/kibana.yml 

Windows： 

- .zip 包安装：同上文中 tar 包安装



## 常见参数优化

​    kibana 的功能已经较为完整，不太需要进行参数上的调整和优化，本节只探讨开启中文显示的方式。

- 同上节中的方式修改对应的配置文件
- 将参数 i18n.locale 设置成 zh-CN 
  - 在 docker-compose.yml 中对应的参数为 I18N_LOCALE



## **常见问题及解决方案**

   kibana节点安装部署过程中经常遇到的一些问题进行分析，并提供一些简单的解决方案。

###### Kibana should not be run as root. Use --allow-root to continue.

- Kibana 和 ES 一样，不能直接通过 root 账号启动 
- 像提示中一样，可以通过添加参数 --allow-root 来启动 
- 完整命令：./bin/kibana --allow-root

###### FATAL Error: Port 5601 is already in use. Another instance of Kibana may be running!

- 5601 端口已被占用

- 修复方式：

  通过 netstat -anp | grep 5601 命令寻找绑定 5601 端口的进程：

  ```shell
  netstat -anp | grep 5601 tcp6 0 0 :::5601 :::* LISTEN 3480/d ocker-proxy-c
  ```

  1. 根据进程信息来决定是否需要关闭已有进程

###### 无法连接到 ES

```shell
log [10:08:28.980] [error][elasticsearch][monitoring] Request error, retrying GET http://localhost:9200/_xpack => connect ECONNREFUSED 127.0.0.1:9200log [10:08:2 8.993] [warning][elasticsearch][monitoring] Unable to revive connection: http://localhost:9200/ log [10:08:28.994] [warning][elasticsearch][monitoring] No living connectionslog [10:08:28. 995] [warning][licensing][plugins] License information could not be obtained from Elasticsear ch due to Error: No Living connections errorlog [10:08:29.016] [warning][monitoring][monit oring][plugins] X-Pack Monitoring Cluster Alerts will not be available: No Living connectionsl og [10:08:29.025] [error][data][elasticsearch] [ConnectionError]: connect ECONNREFUSED 12 7.0.0.1:9200log [10:08:29.059] [error][savedobjects-service] Unable to retrieve version inform ation from Elasticsearch nodes.log [10:08:31.476] [error][data][elasticsearch] [ConnectionErr or]: connect ECONNREFUSED 127.0.0.1:9200
```



- Kibana 无法直接连接 Elasticsearch url，可能有以下原因：
  - 地址配置错误 
  - 当前节点和目标地址中间的网络不通 
  - Elasticsearch 配置的监听地址/端口有误
- 修复方式：
  - 检查该地址/域名是否正确。
  - 通过 curl http://localhost:9200/ 命令测试一下当前节点是否能够正常的连接到目标地址。
  - 调整并调试到正确的连接。

###### 认证失败

```shell
log [10:19:42.007] [error][data][elasticsearch] [security_exception]: missing authenticatio n credentials for REST request [/_nodes?filter_path=nodes.*.version%2Cnodes.*.http.publish_a ddress%2Cnodes.*.ip]log [10:19:42.042] [error][savedobjects-service] Unable to retrieve versi on information from Elasticsearch nodes.log [10:19:42.047] [warning][licensing][plugins] Lice nse information could not be obtained from Elasticsearch due to [security_exception] missi ng authentication credentials for REST request [/_xpack], with { header={ WWW-Authenticate ="Basic realm=\"security\" charset=\"UTF-8\"" } } :: {"path":"/_xpack","statusCode":401,"respon se":"{\"error\":{\"root_cause\":[{\"type\":\"security_exception\",\"reason\":\"missing authenticatio n credentials for REST request [/_xpack]\",\"header\":{\"WWW-Authenticate\":\"Basic realm=\\\ "security\\\" charset=\\\"UTF-8\\\"\"}}],\"type\":\"security_exception\",\"reason\":\"missing authe ntication credentials for REST request [/_xpack]\",\"header\":{\"WWW-Authenticate\":\"Basic re alm=\\\"security\\\" charset=\\\"UTF-8\\\"\"}},\"status\":401}","wwwAuthenticateDirective":"Basic realm=\"security\" charset=\"UTF-8\""} errorlog [10:19:42.050] [warning][monitoring][monitor ing][plugins] X-Pack Monitoring Cluster Alerts will not be available: [security_exception]3.4.1.2 Kibana（本地及 docker) < 162 missing authentication credentials for REST request [/_xpack], with { header={ WWW-Authent icate="Basic realm=\"security\" charset=\"UTF-8\"" } }log [10:19:44.442] [error][data][elasticse arch] [security_exception]: missing authentication credentials for REST request [/_nodes?filter _path=nodes.*.version%2Cnodes.*.http.publish_address%2Cnodes.*.ip]log [10:19:46.941] [erro r][data][elasticsearch] [security_exception]: missing authentication credentials for REST reque st [/_nodes?filter_path=nodes.*.version%2Cnodes.*.http.publish_address%2Cnodes.*.ip]
```



-  Kibana 未配置安全性设置，以至于无法正常连接 Elasticsearch 节点/集群。

- 当 Elasticsearch 节点/集群开启了安全性设置之后，所有的 restful 访问都需要添加认证设置，包括 kibana 的访问。

- 修复方式：

  - （如果没有设置）在 Elasticsearch 集群中选一个节点，运行 bin/elasticsearch-setup-passwords 命令生成所需要账户的密码 

  - 修改配置文件 $KIBANA_HOME/config/kibana.yml 将 kibana 账号（7.x 之后可能会是 kibana_system 账号）对应的密码配置进相关参数中

    ```shell
    `elasticsearch.username: "kibana_system"`
    `elasticsearch.password: "password"`
    ```

- 在配置好安全设置之后初次登陆时需要以管理员（elastic）账号进行登陆，并进行后续的配置和操作 

  

###### Error: Unable to find a match: docker-compose，找不到 docker-compose 对应安装包 

- 可能 yum 仓库中没有最新安装包信息或者精简版系统中没有对应的软件信息

- 修复方式:

  - 把源文件中应用市场的地址替换成中科大:

    ```shell
    sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \ -e 's|^#baseurl=http://mirror.centos.org/centos|baseurl=https://mirrors.ustc.edu.cn /centos|g' \ -i.bak \ /etc/yum.repos.d/CentOS-Base.repo
    ```

    先安装 epel-release （拓展应用市场）

    再进行后续安装



###### Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Isthe docker daemon running?，docker 进程没启动

- docker 安装之后不会自动启动，在未设置之前，服务器重启之后 docker 多半也不会自动重启。
- 修复方式：
  - 启动 docker 进程 systemctl start docker。
  - 设置 docker 随系统启动 systemctl enable docker。

###### Get https://registry-1.docker.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers

访问docker 仓库失败，在某些节点中可能无法直接访问外网进行 docker 镜像的下载

修复方式：

```shell
1. 开启外网访问 
2. （或者）在其他能够访问外网的节点中下载对应镜像 `docker pull kibana:7.10.1` 
3. 把镜像导出为文件 `docker save -o kibana-7.10.1-image.tar docker.io/kibana:7.10.1` 
4. 把导出的文件拷贝到目标机器 `scp kibana-7.10.1-image.tar root@192.168.10.221:/tmp` 
5. 登陆目标机器 `ssh root@192.168.10.221` 1. 导入目标镜像
```



###### Error response from daemon: manifest for kibana:7.9.11 not found: manifest unknown: manifest unknown

- 找不到目标镜像。可能在 docker 仓库中找不到指定版本的镜像 
- 登陆镜像仓库搜索合适版本（http://dockerhub.com/） 
- （或者）通过命令搜索合适的镜像 docker search kibana

