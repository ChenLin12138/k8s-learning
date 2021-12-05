# Docker用WSL启动失败
## Root cause
windows10没有安装wsl，公司网络不让访问微软的wsl网站
## 解决方案
禁用wsl，用Hyper-V启动
C:\Users\user\AppData\Roaming\Docker\settings.json
修改wslEngineEnabled to false

# Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io on 192.168.65.1:53: no such host
## Root cause
我是因为公司网络限制了https://registry-1.docker.io/v2/
## 解决方案
1. 切换到公司外网(中国区网路)
2. 设置-Docker Engine添加registry-mirrors
"registry-mirrors": [
"https://cr.console.aliyun.com"
]
3. 添加国内DNS：设置-Resources-Network-Manual DNS Configration
DNS :114.114.114.114
4. 重启
一般这个有以下问题解决
公司限制-切换到公司外网
国家显示-翻墙，或者添加国内镜像代理

# minickbe 无法安装Ingress插件
```
C:\NotBackedUp\learn\k8s-learning-master\C5>minikube addons enable ingress
* After the addon is enabled, please run "minikube tunnel" and your ingress resources would be available at "127.0.0.1"
  - Using image k8s.gcr.io/ingress-nginx/controller:v1.0.0-beta.3
  - Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
  - Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
* Verifying ingress addon...

X Exiting due to MK_ADDON_ENABLE: run callbacks: running callbacks: [waiting for app.kubernetes.io/name=ingress-nginx pods: timed out waiting for the condition]
*
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                                                                │
│    * If the above advice does not help, please let us know:                                                    │
│      https://github.com/kubernetes/minikube/issues/new/choose                                                  │
│                                                                                                                │
│    * Please run `minikube logs --file=logs.txt` and attach logs.txt to the GitHub issue.                       │
│    * Please also attach the following file to the GitHub issue:                                                │
│    * - C:\Users\user\AppData\Local\Temp\2\minikube_addons_e0e7a0581240141c35e4e8309b248d753611d39a_0.log    │
│                                                                                                                │
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```
根据C:\Users\user\AppData\Local\Temp\2\minikube_addons_e0e7a0581240141c35e4e8309b248d753611d39a_0.log  
log长这样
```
Log file created at: 2021/09/29 17:22:45
Running on machine: CNLT-485CC742
Binary: Built with gc go1.17 for windows/amd64
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0929 17:22:45.426981   16808 out.go:298] Setting OutFile to fd 92 ...
I0929 17:22:45.430565   16808 out.go:345] TERM=,COLORTERM=, which probably does not support color
I0929 17:22:45.430565   16808 out.go:311] Setting ErrFile to fd 96...
I0929 17:22:45.430565   16808 out.go:345] TERM=,COLORTERM=, which probably does not support color
W0929 17:22:45.461158   16808 root.go:291] Error reading config file at C:\Users\user\.minikube\config\config.json: open C:\Users\user\.minikube\config\config.json: The system cannot find the file specified.
I0929 17:22:45.464741   16808 config.go:177] Loaded profile config "minikube": Driver=docker, ContainerRuntime=docker, KubernetesVersion=v1.22.1
I0929 17:22:45.465552   16808 addons.go:65] Setting ingress=true in profile "minikube"
I0929 17:22:45.466156   16808 addons.go:153] Setting addon ingress=true in "minikube"
I0929 17:22:45.471913   16808 host.go:66] Checking if "minikube" exists ...
I0929 17:22:45.498522   16808 cli_runner.go:115] Run: docker container inspect minikube --format={{.State.Status}}
I0929 17:22:47.004987   16808 cli_runner.go:168] Completed: docker container inspect minikube --format={{.State.Status}}: (1.50644s)
I0929 17:22:47.006029   16808 out.go:177] * After the addon is enabled, please run "minikube tunnel" and your ingress resources would be available at "127.0.0.1"
I0929 17:22:47.008094   16808 out.go:177]   - Using image k8s.gcr.io/ingress-nginx/controller:v1.0.0-beta.3
I0929 17:22:47.009852   16808 out.go:177]   - Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
I0929 17:22:47.010947   16808 out.go:177]   - Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
I0929 17:22:47.017401   16808 addons.go:337] installing /etc/kubernetes/addons/ingress-deploy.yaml
I0929 17:22:47.017503   16808 ssh_runner.go:319] scp memory --> /etc/kubernetes/addons/ingress-deploy.yaml (17010 bytes)
I0929 17:22:47.028937   16808 cli_runner.go:115] Run: docker container inspect -f "'{{(index (index .NetworkSettings.Ports "22/tcp") 0).HostPort}}'" minikube
I0929 17:22:48.577133   16808 cli_runner.go:168] Completed: docker container inspect -f "'{{(index (index .NetworkSettings.Ports "22/tcp") 0).HostPort}}'" minikube: (1.5481696s)
I0929 17:22:48.578276   16808 sshutil.go:53] new ssh client: &{IP:127.0.0.1 Port:54933 SSHKeyPath:C:\Users\user\.minikube\machines\minikube\id_rsa Username:docker}
I0929 17:22:48.691304   16808 ssh_runner.go:152] Run: sudo KUBECONFIG=/var/lib/minikube/kubeconfig /var/lib/minikube/binaries/v1.22.1/kubectl apply -f /etc/kubernetes/addons/ingress-deploy.yaml
I0929 17:22:49.396405   16808 addons.go:375] Verifying addon ingress=true in "minikube"
I0929 17:22:49.398000   16808 out.go:177] * Verifying ingress addon...
I0929 17:22:49.439749   16808 kapi.go:75] Waiting for pod with label "app.kubernetes.io/name=ingress-nginx" in ns "ingress-nginx" ...
I0929 17:22:49.461866   16808 kapi.go:86] Found 3 Pods for label selector app.kubernetes.io/name=ingress-nginx
I0929 17:22:49.461866   16808 kapi.go:96] waiting for pod "app.kubernetes.io/name=ingress-nginx", current state: Pending: [<nil>]
I0929 17:22:49.975844   16808 kapi.go:96] waiting for pod "app.kubernetes.io/name=ingress-nginx", current state: Pending: [<nil>]
```
# windows本地启动mongoDB
启动docker容器，并把文件映射到本地
```
docker run -d -p 27017:27017 -v mongodata:/data/db --name=container-mongodb mongo
87e281baf80cc9404f643617bb7c82ae6c3659afe5f1a2887696f9c4dace5e84
```
进入docker容器
```
docker exec -it container-mongodb /bin/bash
```
进入mongodb
```
root@775bf51c3dd6:/data/db# mongo
MongoDB shell version v5.0.3
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
```
查询数据库
```
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
```
创建数据库并插入数据
```
> use test
switched to db test
> db.test.insert({"name":"chenlin"})
WriteResult({ "nInserted" : 1 })
> db.test.find()
{ "_id" : ObjectId("616156634a05be55fa933625"), "name" : "chenlin" }
```

# windows MongoDB报错
```
docker logs container-mongodb
{"t":{"$date":"2021-10-09T09:40:37.562+00:00"},"s":"I",  "c":"NETWORK",  "id":4915701, "ctx":"-","msg":"Initialized wire specification","attr":{"spec":{"incomingExternalClient":{"minWireVersion":0,"maxWireVersion":13},"incomingInternalClient":{"minWireVersion":0,"maxWireVersion":13},"outgoing":{"minWireVersion":0,"maxWireVersion":13},"isInternalClient":true}}}
{"t":{"$date":"2021-10-09T09:40:37.562+00:00"},"s":"I",  "c":"CONTROL",  "id":23285,   "ctx":"main","msg":"Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'"}
{"t":{"$date":"2021-10-09T09:40:37.562+00:00"},"s":"W",  "c":"ASIO",     "id":22601,   "ctx":"main","msg":"No TransportLayer configured during NetworkInterface startup"}
{"t":{"$date":"2021-10-09T09:40:37.562+00:00"},"s":"I",  "c":"NETWORK",  "id":4648601, "ctx":"main","msg":"Implicit TCP FastOpen unavailable. If TCP FastOpen is required, set tcpFastOpenServer, tcpFastOpenClient, and tcpFastOpenQueueSize."}
{"t":{"$date":"2021-10-09T09:40:37.563+00:00"},"s":"W",  "c":"ASIO",     "id":22601,   "ctx":"main","msg":"No TransportLayer configured during NetworkInterface startup"}
{"t":{"$date":"2021-10-09T09:40:37.563+00:00"},"s":"I",  "c":"REPL",     "id":5123008, "ctx":"main","msg":"Successfully registered PrimaryOnlyService","attr":{"service":"TenantMigrationDonorService","ns":"config.tenantMigrationDonors"}}
{"t":{"$date":"2021-10-09T09:40:37.564+00:00"},"s":"I",  "c":"REPL",     "id":5123008, "ctx":"main","msg":"Successfully registered PrimaryOnlyService","attr":{"service":"TenantMigrationRecipientService","ns":"config.tenantMigrationRecipients"}}
{"t":{"$date":"2021-10-09T09:40:37.564+00:00"},"s":"I",  "c":"CONTROL",  "id":4615611, "ctx":"initandlisten","msg":"MongoDB starting","attr":{"pid":1,"port":27017,"dbPath":"/data/db","architecture":"64-bit","host":"4fe85127a59a"}}
{"t":{"$date":"2021-10-09T09:40:37.564+00:00"},"s":"I",  "c":"CONTROL",  "id":23403,   "ctx":"initandlisten","msg":"Build Info","attr":{"buildInfo":{"version":"5.0.3","gitVersion":"657fea5a61a74d7a79df7aff8e4bcf0bc742b748","openSSLVersion":"OpenSSL 1.1.1f  31 Mar 2020","modules":[],"allocator":"tcmalloc","environment":{"distmod":"ubuntu2004","distarch":"x86_64","target_arch":"x86_64"}}}}
{"t":{"$date":"2021-10-09T09:40:37.564+00:00"},"s":"I",  "c":"CONTROL",  "id":51765,   "ctx":"initandlisten","msg":"Operating System","attr":{"os":{"name":"Ubuntu","version":"20.04"}}}
{"t":{"$date":"2021-10-09T09:40:37.564+00:00"},"s":"I",  "c":"CONTROL",  "id":21951,   "ctx":"initandlisten","msg":"Options set by command line","attr":{"options":{"net":{"bindIp":"*"}}}}
{"t":{"$date":"2021-10-09T09:40:37.570+00:00"},"s":"I",  "c":"STORAGE",  "id":22270,   "ctx":"initandlisten","msg":"Storage engine to use detected by data files","attr":{"dbpath":"/data/db","storageEngine":"wiredTiger"}}
{"t":{"$date":"2021-10-09T09:40:37.574+00:00"},"s":"I",  "c":"STORAGE",  "id":22315,   "ctx":"initandlisten","msg":"Opening WiredTiger","attr":{"config":"create,cache_size=481M,session_max=33000,eviction=(threads_min=4,threads_max=4),config_base=false,statistics=(fast),log=(enabled=true,archive=true,path=journal,compressor=snappy),builtin_extension_config=(zstd=(compression_level=6)),file_manager=(close_idle_time=600,close_scan_interval=10,close_handle_minimum=250),statistics_log=(wait=0),verbose=[recovery_progress,checkpoint_progress,compact_progress],"}}
{"t":{"$date":"2021-10-09T09:40:38.082+00:00"},"s":"E",  "c":"STORAGE",  "id":22435,   "ctx":"initandlisten","msg":"WiredTiger error","attr":{"error":1,"message":"[1633772438:82174][1:0x7f9b65426c80], file:WiredTiger.wt, connection: __posix_open_file, 805: /data/db/WiredTiger.wt: handle-open: open: Operation not permitted"}}
{"t":{"$date":"2021-10-09T09:40:38.098+00:00"},"s":"E",  "c":"STORAGE",  "id":22435,   "ctx":"initandlisten","msg":"WiredTiger error","attr":{"error":1,"message":"[1633772438:98526][1:0x7f9b65426c80], file:WiredTiger.wt, connection: __posix_open_file, 805: /data/db/WiredTiger.wt: handle-open: open: Operation not permitted"}}
{"t":{"$date":"2021-10-09T09:40:38.115+00:00"},"s":"E",  "c":"STORAGE",  "id":22435,   "ctx":"initandlisten","msg":"WiredTiger error","attr":{"error":1,"message":"[1633772438:115018][1:0x7f9b65426c80], file:WiredTiger.wt, connection: __posix_open_file, 805: /data/db/WiredTiger.wt: handle-open: open: Operation not permitted"}}
{"t":{"$date":"2021-10-09T09:40:38.116+00:00"},"s":"W",  "c":"STORAGE",  "id":22347,   "ctx":"initandlisten","msg":"Failed to start up WiredTiger under any compatibility version. This may be due to an unsupported upgrade or downgrade."}
{"t":{"$date":"2021-10-09T09:40:38.116+00:00"},"s":"F",  "c":"STORAGE",  "id":28595,   "ctx":"initandlisten","msg":"Terminating.","attr":{"reason":"1: Operation not permitted"}}
{"t":{"$date":"2021-10-09T09:40:38.116+00:00"},"s":"F",  "c":"-",        "id":23091,   "ctx":"initandlisten","msg":"Fatal assertion","attr":{"msgid":28595,"file":"src/mongo/db/storage/wiredtiger/wiredtiger_kv_engine.cpp","line":690}}
{"t":{"$date":"2021-10-09T09:40:38.116+00:00"},"s":"F",  "c":"-",        "id":23092,   "ctx":"initandlisten","msg":"\n\n***aborting after fassert() failure\n\n"}
```
## root cause
windows上面跑的时候权限问题
```
{"t":{"$date":"2021-10-09T09:40:38.115+00:00"},"s":"E",  "c":"STORAGE",  "id":22435,   "ctx":"initandlisten","msg":"WiredTiger error","attr":{"error":1,"message":"[1633772438:115018][1:0x7f9b65426c80], file:WiredTiger.wt, connection: __posix_open_file, 805: /data/db/WiredTiger.wt: handle-open: open: Operation not permitted"}}
```
## 解决方案
创建一个volumn取代路径
```
docker volume create --name=mongodata
docker run -d -p 27017:27017 -v mongodata:/data/db --name=container-mongodb mongo
```
参考文档

[stack overflow](https://stackoverflow.com/questions/61147270/docker-compose-and-mongodb-failed-to-start-up-wiredtiger-under-any-compatibilit/69505759#69505759)

[stack exchange](https://dba.stackexchange.com/questions/186478/mongodb-refuse-to-start-operation-not-permitted)


# minikube 启动MongoDB pod
因为MongoDB需要数据持久化，就是要保证pod重启后数据还在。所以需要把pod中读取文件的位置映射到本地。
```
Error: Error response from daemon: create C/NotBackedUp/mongodb-data: "C/NotBackedUp/mongodb-data" includes invalid characters for a local volume name, only "[a-zA-Z0-9][a-zA-Z0-9_.-]" are allowed. If you intended to pass a host directory, use absolute path
```
## root cause 
minikube其实本质上也是一个容器，它通过docker volume完成了本地系统和minikube容器的映射。我们只需要完成pod到minikube容器的映射即可
## 解决方案 
可以看到minikube本质上是一个跑起来的docker容器
```
docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED       STATUS        PORTS                                                                                                                                  NAMES
2a091bdb1683   kicbase/stable:v0.0.27   "/usr/local/bin/entr…"   2 weeks ago   Up 16 hours   127.0.0.1:50940->22/tcp, 127.0.0.1:50941->2376/tcp, 127.0.0.1:50937->5000/tcp, 127.0.0.1:50938->8443/tcp, 127.0.0.1:50936->32443/tcp   minikube
```
minikube中存储的内容是绑定在dokcer volume
```
docker volume ls
DRIVER    VOLUME NAME
local     minikube
```
我们创建mongodb pod的yaml如下
mongodb-pod-hostpath.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  volumes:
  - name: mongodb-data
    hostPath:
      path: /data/mongodb
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```
所以我们只需要进入minikube的container，创建/data/mongodb目录
```
docker exec -it 2a091bdb1683  /bin/bash
root@minikube:/# cd /data/mongodb/
root@minikube:/data/mongodb# pwd
/data/mongodb
root@minikube:/data/mongodb#
```
创建mongodb pod
```
kubectl create -f mongodb-pod-hostpath.yaml
```
mongodb创建完成
```
C:\NotBackedUp\learn\k8s-learning-master\C6>kubectl get po
NAME      READY   STATUS    RESTARTS   AGE
mongodb   1/1     Running   0          16h
```

# found character that cannot start any token
## root cause
[@ZDMdeMacBook-Pro:3]$ kubectl create -f kubia-manual-with-labels.yaml
error: error parsing kubia-manual-with-labels.yaml: error converting YAML to JSON: yaml: line 6: found character that cannot start any token
## 解决方案
不要在yaml中使用tab

# kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

```
kubectl exec -it fortune-configmap-volume -c html-generator bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
```

## root cause
就是格式不支持了而已，请修改成下面的样子
```
kubectl exec -it fortune-configmap-volume -c html-generator -- bash
root@fortune-configmap-volume:/# exit
exit
```