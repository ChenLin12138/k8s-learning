# 7. ConfigMap和Secret配置应用程序
## 7.1 配置容器化应用程序
配置有以下几种方式:
- 将配置嵌入应用本身
- 命令行参数方式配置应用
- 配置文件话
- 通过环境变量进行配置
k8s提供了ConfigMap这样的资源进行存储配置数据
为社么我们要用这么复杂的配置方式。如果直接将配置写到应用中，里面可能包含了证书，密码，密钥等敏感信息。所以我们会考虑配置文件和应用的分离
本章会讲解以下的文件配置方式:
- 向容器传递命令行参数
- 为每个容器设置自定义环境变量
- 通过特殊类型的卷将配置文件挂载到容器中

## 7.2 向容器传递命令行参数
### 7.2.1 在Dokcer中定义命令行与参数

**了解ENTRYPOINT与CMD**
Dockerfile中两种命令指定分别定义命令与参数这个两个部分：
- ENTRYPOINT定义容器启动时被调用的可执行程序
- CMD指定粗汉帝给ENTRYPOINT的参数
尽管可以直接使用CMD命令镜像运行时向要执行的命令，正确的做法依旧是借助ENTRYPOINT指令，仅仅用CMD指定所需的默认参数，这样镜像可以直接运行，无需添加任何参数。
镜像直接运行，而不添加任何参数的方法
```
docker run <image>
```

或者添加一些参数，覆盖Dockerfile中任何由CMD指定的默认参数
```
docker run <image> <args>
```

Dockerfile
```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node","app.js"]
```

build docker 镜像
```
docker build -t kubia .
```

app.js
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

启动image kubia
```
docker run kubia
```

进入容器查看进程情况
ps x表示显示所有进程，不以终端机区分
```
docker ps
CONTAINER ID   IMAGE     COMMAND         CREATED          STATUS          PORTS     NAMES
5ad8c3b20bb1   kubia     "node app.js"   13 minutes ago   Up 13 minutes             vibrant_taussig
docker exec -it 5ad8c3b20bb1 bash
root@5ad8c3b20bb1:/# ps x
  PID TTY      STAT   TIME COMMAND
    1 ?        Ssl    0:00 node app.js
   31 pts/0    Ss     0:00 bash
   39 pts/0    R+     0:00 ps x
```
这里可以看到 node app.js是一个单独的进程

Dockerfile-shell
```
FROM node:7
ADD app.js /app.js
ENTRYPOINT node app.js
```
和上面使用一样的app.js
使用Dockerfile-shell build image kubia-shell 
```
docker build -t kubia-shell . -f Dockerfile-shell
docker run kubia-shell
docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS     NAMES
8d30c3d19acc   kubia-shell   "/bin/sh -c 'node ap…"   14 seconds ago   Up 12 seconds             inspiring_khayyam
docker exec -it 8d30c3d19acc bash
root@8d30c3d19acc:/# ps x
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /bin/sh -c node app.js
    8 ?        Sl     0:00 node app.js
   14 pts/0    Ss     0:00 bash
   21 pts/0    R+     0:00 ps x
```
可以看出利用shell启动的在command里的命令是这样的/bin/sh -c node app.js

**可配置化fortune镜像中的间隔参数**
fortuneloop.sh脚本
```
#!/bin/bash
trap "exit" SIGINT

INTERVAL=$1
echo Configured to generate new fortune every $INTERVAL seconds

mkdir -p /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

Dockerfile-fortune
其中CMD["10"]为可执行程序的默认参数
```
FROM ubuntu:latest
RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh
ENTRYPOINT ["/bin/fortuneloop.sh"]
CMD ["10"]
```

创建docker image
```
docker build -t fortune:args . -f Dockerfile-fortune
```

下面问题是因为我使用的windows平台的编辑器，回车换行是crlf,需要切换为lf来解决这个问题，可以参考文档
https://blog.csdn.net/lsqtzj/article/details/120619305
```
docker run -it fortune:args
standard_init_linux.go:228: exec user process caused: no such file or directory
```

重新build image 
```
docker build -t fortune:args . -f Dockerfile-fortune
```

查看下面的结果,也可可以看出默认参数是10秒，我们可以传入自己定义的参数15秒
这里表达的参数是从启动docker的命令中传入的
```
docker run fortune:args
Configured to generate new fortune every 10 seconds
Fri Nov 26 06:00:48 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:00:58 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:01:08 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:01:18 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:01:28 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:01:38 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:01:48 UTC 2021 Writing fortune to /var/htdocs/index.html

docker run fortune:args 15
Configured to generate new fortune every 15 seconds
Fri Nov 26 06:02:05 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:02:20 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:02:35 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:02:50 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:03:05 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:03:20 UTC 2021 Writing fortune to /var/htdocs/index.html
Fri Nov 26 06:03:35 UTC 2021 Writing fortune to /var/htdocs/index.html
```
### 7.2.2 在Kubernates中覆盖命令中的参数
在k8s中定义容器时，镜像的entrypoint和cmd均可以被覆盖，仅需在容器定义中设置属性command和args的值
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: luksa/fortune:args
    args: ["2"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

登录到po中去，查看输出结果
```
kubectl exec fortune2s  -it bash
cat /var/htdocs/index.html
Q:      What's a light-year?
A:      One-third less calories than a regular year.
```
## 7.3 为容器设置环境变量
容器通常会使用环境变量作为配置资源，下面我们通过环境变量配置fortune镜像中的间隔值
fortuneloop-env.sh
其实我们用的别人的容器，根本不需要这个
```
#!/bin/bash
trap "exit" SIGINT

echo Configured to generate new fortune every $INTERVAL seconds

mkdir -p /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

fortune-pod-env.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```
可以感受到现在/var/htdocs/index.html 文件30秒更新一次

### 7.3.2 在环境变量值中引用其他环境变量
```
env:
- name: FIRST_VAR
  value: "foo"
- name: SECOND_VAR
  value: "$(FIRST VAR)bar" // foobar
```

### 7.3.3 了解硬编码环境变量的不足之处
接下来的事情是我们想用ConfigMap解耦配置

## 7.4 利用ConfigMap解耦配置
### 7.4.1 ConfigMap介绍
### 7.4.2 创建ConfigMap
**使用指令创建Configmap**
```
kubectl create configmap fortune-config --from-literal=sleep-interval=25
configmap/fortune-config created
```
通过configration map创建多个条目
```
kubectl create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two
configmap/myconfigmap created
```
查看configmap的定义
```
kubectl get configmap fortune-config -o yaml
apiVersion: v1
data:
  sleep-interval: "25"
kind: ConfigMap
metadata:
  creationTimestamp: "2021-11-26T08:39:03Z"
  name: fortune-config
  namespace: default
  resourceVersion: "725103"
  uid: 22788a04-ac77-4075-82b7-957a365ff83b
```
### 7.4.3 给容器传递ConfigMap条目作为环境变量
定义env INTERVAL来自configMapKeyRef来定义pod中环境变量值的来源
fortune-pod-env-configmap.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: library/nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir:
      medium: Memory
```

创建pod
```
kubectl create -f fortune-pod-env-configmap.yaml
pod/fortune-env-from-configmap created
```
检查输出结果
```
root@fortune-env-from-configmap:/# cat /var/htdocs/index.html
Tonight you will pay the wages of sin; Don't forget to leave a tip.
root@fortune-env-from-configmap:/# cat /var/htdocs/index.html
```
### 7.4.4 一次性传递ConfigMap的所有条目作为环境变量
以下结果会给configMap中的所有变量统一添加前缀CONFIG_
```
  containers:
  - image: luksa/fortune:env
    envFrom:
    prefix: CONFIG_
      configMapRef:
          name: fortune-config
```

### 7.4.5 传递ConfigMap条目作为命令行参数
fortune-pod-args-configmap.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap
spec:
  containers:
  - image: luksa/fortune:args
    env:
    - name: INTERVAL
      valueFrom: 
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
``` 
### 7.4.6 使用ConfigMap卷将条目暴露为文件
**创建ConfigMap**
创建一个文件夹confogmap-files/
下面包含以下2个文件
my-nginx-config.conf 和 sleep-interval 

my-nginx-config.conf
```
server {
    listen              80;
    server_name         www.kubia-example.com;

    gzip on;
    gzip_types text/plain application/xml;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

}
```

sleep-interval 
```
25
```

从文件夹创建ConfigMap:
```
kubectl create configmap fortune-config --from-file=configmap-files
```
查询从文件创建的configmap的yaml格式定义
可以看出下面有两个条目:my-nginx-config.conf和sleep-interval
条目的键名和文件名相同，接下来我们准备在pod的容器中，使用该ConfigMap
```
kubectl get configmap fortune-config -o yaml
apiVersion: v1
data:
  my-nginx-config.conf: "server {\r\n    listen              80;\r\n    server_name
    \        www.kubia-example.com;\r\n\r\n    gzip on;\r\n    gzip_types text/plain
    application/xml;\r\n\r\n    location / {\r\n        root   /usr/share/nginx/html;\r\n
    \       index  index.html index.htm;\r\n    }\r\n\r\n}"
  sleep-interval: "25"
kind: ConfigMap
metadata:
  creationTimestamp: "2021-11-29T14:04:46Z"
  name: fortune-config
  namespace: default
  resourceVersion: "924248"
  uid: a58b4721-9f25-4970-b583-13a2902c1a2a
```

**在卷内使用ConfigMap的条目**
























