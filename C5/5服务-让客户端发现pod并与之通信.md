# 5. 服务：让客户端发现pod并与之通信
## 5.1 介绍服务
k8s的服务是一种为一组功能相同的pod提供单一不变的接入点的资源。
### 5.1.1 创建服务
**通过kubectl expose创建服务**
**通过yaml描述文件来创建服务**
通过下面文件来创建 service kubia-svc.yaml
port:80指定服务可用的端口
targetPort:8080服务将连接转发到的容器端口
selector:app:kubia 具有app=kuiba标签的pod都属于该服务
这里的说道说道:这里表达的是创建一个kubia的服务，他在80端口接受请求，并将请求链路到具有标签选择器app:kubia的pod的8080端口上。
```
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```
创建服务
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl create -f kubia-svc.yaml
service/kubia created
```
**检测新的服务**
**服务查询**
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   5d18h
kubia        ClusterIP   10.98.177.109   <none>        80/TCP    21s
```
这里还需要启动一个pod，我们启动一个ReplicaSet来控制一个pod吧
kubia-replicaset.yaml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: scorpionchenlin/kubia
```
启动这个ReplicaSet
```
kubectl create -f kubia-replicaset.yaml
```
列表中显示分配给服务的ip是集群内部的地址(10.98.177.109)，只能在集群内部可以被访问到。服务的主要目标是使集群的内部其它pod可以访问当前这组pod，但是有时也需要对外暴露服务。
**从内部集群测试服务**
根据service的内部ip10.98.177.109,测试pod
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec kubia-6kds9 -- curl -s http://10.98.177.109
You've hit kubia-gqb8r
```
> 为什么是双斜杠(--)
--表示kubectl命令的结束，在两个横杠之后的内容是指在pod内部需要执行的命令。如果要执行的命令并没有以横杠开始的参数。横杠也不是必须的。因为这个命令中有个-s，所以我们必须要--.
比如下面例子也是成立的,在curl中-s表示将不输出错误信息和进度信息。
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec kubia-6kds9  curl  http://10.98.177.109
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    23    0    23    0     0  44660      0 --:--:-- --:--:-- --:--:-- 23000
You've hit kubia-gqb8r
```
其实我们是可以看到，我们登录了一个rs管理的3个pod的任意一个，kubia-6kds9，kubia-gqb8r，kubia-m2nwb. 然后在某一个pod的内部向service(10.98.177.109) 发送curl请求。然后这个请求被转发到了service管理任意一个pod上进行处理。
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get po
NAME          READY   STATUS    RESTARTS   AGE
kubia-6kds9   1/1     Running   0          17m
kubia-gqb8r   1/1     Running   0          17m
kubia-m2nwb   1/1     Running   0          17m
```
**配置服务器上的花花亲和性**
正常情况下每一次请求可能会被service分配到不同的pod上。如果希望指定客户端产生的请求都派发到指定的pod上那么需要使用下面参数
sessionAffinity: ClientIP
```
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP
```
但是你需要注意的是，这里的亲和度是基于ip来的，并不提供cookie级别的支持。这是因为k8s不是工作在http层面上的应用。而cookie信息是包含在http层面上的。这就是k8s不能基于cookie来做亲和度。
**同一个服务暴露多个端口**
一个服务可以暴露多个端口。一个pod也可以监听多个端口。我们举个例子：
比如我们的http监听8080端口，https监听8443端口
一个服务从80端口转发数据到8080端口，443转发请求到8443端口，我们看下面的配置

```
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  selector:
    app: kubia
```
下面https不能工作，是因为我们创建的kubia的镜像不监听8443端口
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec kubia-6kds9 -- curl -s http://10.106.255.46
You've hit kubia-gqb8r

C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec kubia-6kds9 -- curl -s https://10.106.255.46
command terminated with exit code 7
```
配置在这里
```
const http = require('http');
const os = require('os');
console.log("Kubia server starting...");
var handler = function(request, response) {
console.log("Received request from " + request.connection.remoteAddress);
response.writeHead(200);
response.end("You've hit " + os.hostname() + "\n");
};
var www = http.createServer(handler);
www.listen(8080);
```
> 标签选择器将用于整个服务，不能对每个端口单独配置，如果不同的pod有不同的端口映射关系，需要创建两个服务。

**使用命名端口**
cat kubia-rc-named-port.yaml
命名端口号是用http替代了之前的80,使用https替代了之前的443
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: samzhang80/kubia
        ports:
        - name: http
          containerPort: 8080
        - name: https
          containerPort: 8443
```
这样的好处是，当我们更改端口号的时候，也不用再更改service的spec,只需要修改pod中的端口号。

### 5.1.2 服务发现
**通过环境变量发现服务**

```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec kubia-4hfmc env
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-4hfmc
KUBIA_PORT_443_TCP_PROTO=tcp
KUBIA_PORT_443_TCP_PORT=443
KUBIA_PORT_443_TCP_ADDR=10.106.255.46
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBIA_SERVICE_PORT=80
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBIA_SERVICE_PORT_HTTPS=443
KUBIA_PORT=tcp://10.106.255.46:80
KUBIA_PORT_80_TCP_PROTO=tcp
KUBIA_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBIA_PORT_80_TCP_ADDR=10.106.255.46
KUBIA_PORT_443_TCP=tcp://10.106.255.46:443
KUBERNETES_SERVICE_PORT=443
KUBIA_PORT_80_TCP=tcp://10.106.255.46:80
KUBERNETES_PORT_443_TCP_PORT=443
KUBIA_SERVICE_HOST=10.106.255.46
KUBIA_SERVICE_PORT_HTTP=80
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=7.10.1
YARN_VERSION=0.24.4
HOME=/root
```

```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          5d22h
kubia        ClusterIP   10.106.255.46   <none>        80/TCP,443/TCP   46m
```
从上面的查询可以看出:
KUBIA_SERVICE_HOST=10.106.255.46
KUBIA_SERVICE_PORT=80
表示kubia服务的ip号和端口
在本章最开始的例子中，前端pod需要后端pod的时候，可以通过名为back-database的服务将后端pod暴露出来，然后前端pod通过通过环境变量backend_database_service_host和backend_database_service_port去获取ip地址和端口信息。
**通过DNS发现服务**
下面有一个coredns-78fcd69978-dft89的pod,我没有看到有叫cube-dns的pod
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get po --namespace kube-system
NAME                               READY   STATUS    RESTARTS      AGE
coredns-78fcd69978-dft89           1/1     Running   2 (23h ago)   11d
etcd-minikube                      1/1     Running   2 (23h ago)   11d
kube-apiserver-minikube            1/1     Running   2 (23h ago)   11d
kube-controller-manager-minikube   1/1     Running   2 (23h ago)   11d
kube-proxy-ff7l7                   1/1     Running   2 (23h ago)   11d
kube-scheduler-minikube            1/1     Running   2 (23h ago)   11d
storage-provisioner                1/1     Running   5 (23h ago)   11d
```
**通过FQDN连接服务**
FQDN全限定域名，
好吧，他建议我们别这样用，我我们换个命令进入一个pod的bash
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec -it kubia-4hfmc bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
```
新的进入一个pod bash的命令
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec -it kubia-4hfmc -- bash
root@kubia-4hfmc:/#
```
使用全限定域名来访问服务,下面命令中
kubia是服务名
default是命名空间
svc.cluster.local是所有集群本地服务名称重使用的可配置集群域的后缀
```
root@kubia-4hfmc:/# curl http://kubia.default.svc.cluster.local
You've hit kubia-f8h67
```
当我们发起和接受访问的pod在同一个命名空间下，可以使用服务名来指代服务
```
root@kubia-4hfmc:/# curl http://kubia
You've hit kubia-f8h67
```
这不就是集群内部的dns吗

**无法ping通服务ip的原因**
我们发现curl可以工作，但是ping不行。这是因为服务器集群ip是一个虚拟ip,只有在与端口结合的时候才有意义。所以调试异常的时候不要用ping
```
root@kubia-f8h67:/# ping kubia
PING kubia.default.svc.cluster.local (10.106.255.46): 56 data bytes
```
## 5.2 连接集群外部的服务
### 5.2.1 介绍服务endpoint
服务并不是直接与pod连接的，他们只有有一种叫endpoint的资源。
用于创建endpoint列表的服务pod选择器
Selector:app=kubia
代表服务endpoint的pod的ip和端口列表
Endpoints:172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl describe svc kubia
Name:              kubia
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=kubia
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.106.255.46
IPs:               10.106.255.46
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080
Port:              https  443/TCP
TargetPort:        8443/TCP
Endpoints:         172.17.0.10:8443,172.17.0.8:8443,172.17.0.9:8443
Session Affinity:  None
Events:            <none>
```
获取endpoints的基本信息
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get endpoints kubia
NAME    ENDPOINTS                                                      AGE
kubia   172.17.0.10:8443,172.17.0.8:8443,172.17.0.9:8443 + 3 more...   126m
```
尽管在service层级的定义中定义了pod资源选择器，但是外部请求传入service转发到pod的时候并不是由选择器完成的。选择器用于构建ip和端口列表，然后将他们储存在Endpoint中。当客户端连接到服务时，服务器代理选择这些ip和端口对中的一个，
将请求转发。
### 5.2.2 手动配置服务endpoint
如果创建了不包含pod选择器的服务，k8s将不会创建EndPoint资源.
**创建没有选择器的服务**
服务名称和Endpoint对象的名字相匹配，服务中没有定义选择器
external-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```
由于上面的yaml中没有定义选择器，所以这个服务中没有po资源，我们现在来手动创建资源
external-service-endpoints.yaml
将服务连接重定向到endpoint的ip地址是11.11.11.11和22.22.22.22,端口是80，这里注意end-point和service的名字是相同的
```
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  // 挂载到Service后可实现负载均衡
  - addresses:
    - ip: 11.11.11.11
    - ip: 22.22.22.22
    ports:
    - port: 80
```
创建外部服务的endpoint
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl create -f external-service-endpoints.yaml
endpoints/external-service created
```
外部服务被创建起来了
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
external-service   ClusterIP   10.106.122.93   <none>        80/TCP           13m
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP          6d
kubia              ClusterIP   10.106.255.46   <none>        80/TCP,443/TCP   176m
```

### 5.2.3 为外部服务创建别名
除了手动配置endpoint来代替公开的外部服务器外，还有一种更简单的方案，通过完全限定名访问外部服务
```
apiVersion: v1
kind: Service
metadata:
  name: external-service-v2
spec:
  type: ExternalName
  externalName: www.baidu.com
  ports:
  - port: 80
```
创建外部别名服务
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl create -f external-service-externalname.yaml
service/external-service-v2 created
```
我还是没有成功的连接到外部网络
最终不直到是网络的问题，还是百度的问题，还是我的问题，没有成功。
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl exec -it kubia-4hfmc -- bash
root@kubia-4hfmc:/# curl http://external-service
^C
root@kubia-4hfmc:/# curl http://external-service-externalname
curl: (6) Could not resolve host: external-service-externalname
```

## 5.3将服务暴露给外部客户端
现在我们聊聊如何把服务暴露给外部
下面三种方式:
- 将服务的类型设置成NodePort，每个集群节点都会在节点上开打一个集群端口，对于NodePort服务，每个集群在节点本身上打开一个端口。并将该端口上接受到的流量重定向到基础服务。
- 将服务设置成LoadBalance模式。这是Nodeport的扩展模式，这使得妇科可以通过一个专用的负载均衡器来访问，这个是由K8S提供的云基础设施提供的。
- 创建一个Ingress资源，这是一个完全不同的机制，通过一个ip地址公开多个服务。他运行为http层。

## 5.3.1 使用NodePort类型服务
这种模式剋将一组pod公开给外部客户，通过创建NodePort， k8s可以在所有节点上保留一个端口，所有节点上都使用相同的端口号。并将传入的连接转发给服务部分的pod.
**创建NodePort类型的服务**
kubia-svc-nodeport.yaml
type是NodePort,nodePort30123是节点上保留端口号，表示通过集群节点的30123可以访问该服务
80表示服务集群ip的端口号
```
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl create -f kubia-svc-nodeport.yaml
service/kubia-nodeport created
```
查看服务类型
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get svc
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
external-service                ClusterIP      10.106.122.93   <none>          80/TCP           22h
external-service-externalname   ExternalName   <none>          www.baidu.com   80/TCP           22h
kubernetes                      ClusterIP      10.96.0.1       <none>          443/TCP          6d23h
kubia                           ClusterIP      10.106.255.46   <none>          80/TCP,443/TCP   25h
kubia-nodeport                  NodePort       10.100.90.155   <none>          80:30123/TCP     13s
```
按照书籍的介绍:
服务ip:80可以访问
节点ip:30123可以访问
我们在external-ip中没有看到nodes，但是如果看到了，他表示可以通过任何集群节点的ipd地址访问。
最后minikube 使用 minikube service <service-name>访问NodePort服务
```
C:\NotBackedUp\learn\k8s-learning-master\C5>minikube service kubia-nodeport
! Executing "docker container inspect minikube --format={{.State.Status}}" took an unusually long time: 2.2384847s
* Restarting the docker service may improve performance.
|-----------|----------------|-------------|---------------------------|
| NAMESPACE |      NAME      | TARGET PORT |            URL            |
|-----------|----------------|-------------|---------------------------|
| default   | kubia-nodeport |          80 | http://192.168.49.2:30123 |
|-----------|----------------|-------------|---------------------------|
* Starting tunnel for service kubia-nodeport.
|-----------|----------------|-------------|------------------------|
| NAMESPACE |      NAME      | TARGET PORT |          URL           |
|-----------|----------------|-------------|------------------------|
| default   | kubia-nodeport |             | http://127.0.0.1:54450 |
|-----------|----------------|-------------|------------------------|
```
我们现在的效果是,整个互联网可以通过任何节点上的30123端口访问到我们的pod
我尝试用节点去访问，也失败了。192.168.49.2:30123不能成功
```
C:\NotBackedUp\learn\k8s-learning-master\C5>kubectl get no -o=jsonpath='{.items[*].status.addresses[*].address}'
'192.168.49.2 minikube'
```
### 5.3.2 通过负载均衡器将服务暴露出来
kubia-svc-loadbalancer.yaml，创建kubia-loadbalancer服务
```
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
  selector:
    app: kubia
```

```
kubectl create -f kubia-svc-loadbalancer.yaml
```
minikube可以使用下面命令访问
```
C:\NotBackedUp\learn\k8s-learning-master\C5>minikube service kubia-loadbalancer
|-----------|--------------------|-------------|---------------------------|
| NAMESPACE |        NAME        | TARGET PORT |            URL            |
|-----------|--------------------|-------------|---------------------------|
| default   | kubia-loadbalancer | http/80     | http://192.168.49.2:30462 |
|-----------|--------------------|-------------|---------------------------|
* Starting tunnel for service kubia-loadbalancer.
|-----------|--------------------|-------------|------------------------|
| NAMESPACE |        NAME        | TARGET PORT |          URL           |
|-----------|--------------------|-------------|------------------------|
| default   | kubia-loadbalancer |             | http://127.0.0.1:58285 |
|-----------|--------------------|-------------|------------------------|
* Opening service default/kubia-loadbalancer in default browser...
! Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```
在使用Loadbalancer的时候，即使没有绘画亲和性设置，同一个浏览器发起的请求也会由同一个pod处理。因为浏览器使用了keep-alive连接

### 5.3.3 了解外部连接的特性
**了解并防止不必要的网络跳数**
当外部客户端通过节点端口连接到服务的时候，随机选择的Pod并不一定在接受连接的同一节点上运行，可能需要额外的网络跳转才能达到pod.这种情况是不符合预期的，可以使用下面的配置来避免这样的现象。
```
spec.externalTrafficPolicy: local
```
在有了这个设置后，如果本地没有pod存在则连接挂起.因此要保证负载均衡器将连接转发给至少具有一个pod的节点。
如果没有这个注解，那么他们会针对全局的pod随机转发。
## 5.4 通过Ingress暴露服务
**为什么需要Ingress**
每一个LoadBalancer服务都需要有自己的ip，而Ingress只需要一个公网IP就可以让很多服务提供访问。
Ingress的网络栈的应用操作可以提供cookie的会话亲和性功能。
在minikube上启动Ingress的扩展功能
minikube查看Ingress的扩展功能
```
C:\NotBackedUp\learn\k8s-learning-master\C5>minikube addons list
|-----------------------------|----------|--------------|-----------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |      MAINTAINER       |
|-----------------------------|----------|--------------|-----------------------|
| ambassador                  | minikube | disabled     | unknown (third-party) |
| auto-pause                  | minikube | disabled     | google                |
| csi-hostpath-driver         | minikube | disabled     | kubernetes            |
| dashboard                   | minikube | enabled ✅   | kubernetes            |
| default-storageclass        | minikube | enabled ✅   | kubernetes            |
| efk                         | minikube | disabled     | unknown (third-party) |
| freshpod                    | minikube | disabled     | google                |
| gcp-auth                    | minikube | disabled     | google                |
| gvisor                      | minikube | disabled     | google                |
| helm-tiller                 | minikube | disabled     | unknown (third-party) |
| ingress                     | minikube | disabled     | unknown (third-party) |
| ingress-dns                 | minikube | disabled     | unknown (third-party) |
| istio                       | minikube | disabled     | unknown (third-party) |
| istio-provisioner           | minikube | disabled     | unknown (third-party) |
| kubevirt                    | minikube | disabled     | unknown (third-party) |
| logviewer                   | minikube | disabled     | google                |
| metallb                     | minikube | disabled     | unknown (third-party) |
| metrics-server              | minikube | disabled     | kubernetes            |
| nvidia-driver-installer     | minikube | disabled     | google                |
| nvidia-gpu-device-plugin    | minikube | disabled     | unknown (third-party) |
| olm                         | minikube | disabled     | unknown (third-party) |
| pod-security-policy         | minikube | disabled     | unknown (third-party) |
| portainer                   | minikube | disabled     | portainer.io          |
| registry                    | minikube | disabled     | google                |
| registry-aliases            | minikube | disabled     | unknown (third-party) |
| registry-creds              | minikube | disabled     | unknown (third-party) |
| storage-provisioner         | minikube | enabled ✅   | kubernetes            |
| storage-provisioner-gluster | minikube | disabled     | unknown (third-party) |
| volumesnapshots             | minikube | disabled     | kubernetes            |
|-----------------------------|----------|--------------|-----------------------|
```
启动
下面本地没有成功，所以在minikube模拟平台上测试的
```
minikube addons enable ingress
```
查询ingress的po
```
kubectl get po --all-namespaces
```

### 5.4.1 创建Ingress资源
kubia-ingress.yaml
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```
创建一个ingress资源
```
$ kubectl create -f kubia-svc-loadbalancer.yaml
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/kubia created
```
### 5.4.2 通过Ingress访问服务
获取Ingress的ip地址
```
$ kubectl get ing
NAME    CLASS    HOSTS               ADDRESS        PORTS   AGE
kubia   <none>   kubia.example.com   172.17.0.111   80      51s
```
在hosts中配置ingress域名映射
**通过Ingress访问pod**
```
curl http://kubia.example.com
```




