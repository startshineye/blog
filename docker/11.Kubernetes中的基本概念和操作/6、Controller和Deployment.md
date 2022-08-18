
### 1、k8s的架构  
   我们从之前了解了我们的k8s架构，如下:
![](../images/57.png)  

注意:
1、k8s集群架构分为:Master节点跟普通的Node节点。
2、Master节点分为:etcd-用于存储  API Server-提供给对外通信:kubectl会连接这个API Server，然后控制整个集群。 Scheduler:比如说我们创建资源pod的时候,这个
集群决定这个pod放在哪个node上是由此组件进行调度的(调度时候会使用一些算法:比如分析哪台机器资源最优等)。以及Controller

### 2、Controller
   Controller到底是用于实现什么目的的呢？我们从程序的角度讲解，一个controller到底要做一些什么事？如下函数调用的脚本 分析功能：
![](../images/58.png)  

注意：
1、我们定义了两个状态:desired--期望的状态  current-当前实际的状态；我们有可能期望他是:运行的状态，但是实际上可能是不运行的状态。
2、我们的controller主要是做：makeChanges的动作。让我们的服务从current状态达到期望状态。


### 3、Deployment
A Deployment provides declarative updates for Pods and ReplicaSets.  
You describe a desired state in a Deployment, and the Deployment Controller changes 
the actual state to the desired state at a controlled rate. 
You can define Deployments to create new ReplicaSets, 
or to remove existing Deployments and adopt all their resources with new Deployments.


```renderscript
Note: Do not manage ReplicaSets owned by a Deployment. 
Consider opening an issue in the main Kubernetes repository if your use case is not covered below.
```

[](../images/57.png) 
场景描述:
   基于k8s的架构，比如我们通过了Scheduler调度器来创建了在以上第一个Node上；如果这个时候假如
第一个Node的linux server已经宕机了，那么这个机器上创建的Pod也随之死掉。所以Pod并不是一个稳定的存在。
如果不稳定，我们肯定不会用他，所以在实际中我们不会直接使用Pod,而是使用Deployment。
因为Deployment描述了一个期望的状态，然后我们的Controller会监听尽可能的使我们的状态等于我们的期望状态。

比如我们定义了一个状态，此状态下会有一个Pod，然后里面可能包含多个容器。假如我们的Pod所在的机器宕机了，Pod销毁
此时Controller就会启作用，在另一个地方启动这个Pod。  






















