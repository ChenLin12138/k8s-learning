# 15 Pod与集群节点的自动伸缩
## 15.1 Pod的横向自动伸缩
伸缩有两个方向，一个是水平扩展，一个是垂直扩展。水平就是+pod的数量，垂直就是增大CPU的性能。
### 15.1.1 了解自动伸缩过程
**获取pod度量**
**计算所需的pod资源**
**更新被伸缩资源的副本数**
Autoscaler可以控制以下资源
- Deploymeny
- ReplicaSet
- ReplicationController
- StatefulSet
### 15.1.2 基于CPU使用率进行自动伸缩
deployment.yaml
下面的设置副本数为3，每个pod请求100毫核的CPU
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m
```
在Deployment之后，为了给它的pod启用横向自动伸缩，需要创建一个HorizationpodAutoscaler(HPA)对象，并把它指向该Deployment。下面语句为你创建HPA。他设置了CPU使用率30%，并制定了最小和最大的副本数量。autoscale会动态调整deployment的数量，直到CPU使用率接近30%。当然他不会小于1，>5个。
这里的30%指的是单个pod的cpu使用率不超过30%
```
k autoscale deployment kubia --cpu-percent=30 --min=1 --max=5
```
那么在设置了之后，我们能看到的是，当没有请求的时候。pod的使用率几乎为0%。那么他会不断的收缩最后收缩到只剩一个pod。当请求变多之后，单个cpu的负载增加。pod会随着负债增加而增加pod数量。

容器CPU的使用率是它实际的CPU使用除以它CPU请求。CPU请求标记的事他的最少资源量。所以一个容器的CPU使用率比CPU请求的高。
**最终使用了4个POD的原因**
一个副本的时候CPU使用率108%。目标使用率30。108/30=3.6.那么我们4个pod非常合适。

### 15.1.3 基于内存使用进行自动伸缩
基于内存自动伸缩比基于CPU的困难的多。主要是扩容后原油的pod如果是要释放内存，需要应用自己进行释放。而不能由于操作系统代劳。这一点CPU能够做到。

### 15.1.4 基于其它自定义度量进行自动伸缩
**了解Pods度量类型**
Pod类型用来引用任何其他种类与pod直接相关的度量。例如每秒查询次数，消息队列中的消息数量。
**了解Object度量类型**
Autoscaler可以根据Object的数量个数来决定Pod的自动伸缩。

### 15.1.5 确定哪些度量适合用于自动伸缩
副本的增加能够导致被观测度量平均值的线性下降的度量才适合自动伸缩。

### 15.3.1 Cluster Autoscaler介绍
**从云端基础架构请求新节点**
如果在一个pod创建后，Scheduler无法将它调度到任何一个已有的节点，一个新的节点就会被创建。 Cluster Autoscaler会注意到这类Pod.并请求云服务提供者启动一个新的节点。但是这么做之前。他会检查新节点又没有可能能容纳这个pod.如果不能的话，他就没必要启动一个新的节点了。

### 15.3.2 启用Cluster Autoscaler
集群自动伸缩的以下云服务提供商可用：
GKE
GCE
AWS
MA