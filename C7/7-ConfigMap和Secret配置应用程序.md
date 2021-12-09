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

下面的配置表示ngix会压缩传递给客户端的响应
my-nginx-config.conf
```
# server其实是内嵌到http块中的，协议级别配置
# 服务级别配置，一个http中可以有多个server。
server {
    # 监听端口
    listen              80;
    # 监听地址
    server_name         www.kubia-example.com;
    
    # 开启gzip压缩
    gzip on;
    
    # 设置压缩类型
    gzip_types text/plain application/xml;

    # 请求级别配置
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
命令中-o 表示output, yaml表示输出格式
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
fortune-pod-configmap-volume.yaml
这个yaml文件指定在pod中创建2个contrainer: html-generator和web-server，并且指定web-server的config来自于configMap，并且绑定到/etc/nginx/conf.d下面
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
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
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: config
      mountPath: /tmp/whole-fortune-config-volume
      readOnly: true
    ports:
      - containerPort: 80
        name: http
        protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
```
**检查Nginx是否使用被挂载的配置文件**
```
kubectl port-forward fortune-configmap-volume 8080:80 &
[1] 10117
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```
查看挂载内容
```
curl -H "Accept-Encoding: gzip" -I localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.21.4
Date: Sun, 05 Dec 2021 07:52:56 GMT
Content-Type: text/html
Last-Modified: Sun, 05 Dec 2021 07:52:54 GMT
Connection: keep-alive
ETag: W/"61ac6fd6-dc"
Content-Encoding: gzip
```

因为pod fortune-configmap-volume 中其实是有2个应用的，而且这两个应用分别运行在不同的容器中。
两个容器为：
html-generator
web-server
在定义的时候，我们是先定义html-generator的，所以在用kubectl exec -it的时候,我们想看web-server的内容，我们需要用-c 来指定容器。下面命令我可以看到绑定成功了。
```
kubectl exec -it fortune-configmap-volume -c web-server -- ls /etc/nginx/conf.d
my-nginx-config.conf  sleep-interval
```

对于下面不能用--bash登录web-server的情况，我们可以考虑使用：/bin/sh来代替
```
kubectl exec -it fortune-configmap-volume -c web-server -- bash
OCI runtime exec failed: exec failed: container_linux.go:380: starting container process caused: exec: "bash": executable file not found in $PATH: unknown
command terminated with exit code 126
```
/bin/sh的处理方式
```
kubectl exec -it fortune-configmap-volume -c web-server -- /bin/sh
/ # cd /etc/nginx
/etc/nginx # cd conf.d
/etc/nginx/conf.d # ls -l
total 0
lrwxrwxrwx    1 root     root            27 Dec  5 07:48 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root            21 Dec  5 07:48 sleep-interval -> ..data/sleep-interval
```
**卷内暴露指定的ConfigMap条目**
上面的内容暴露了configmap中所有的条目，其实我们是有办法只暴露configMap卷中部分的条目
fortune-pod-configmap-volume-with-items.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-with-items
spec:
  containers:
  - image: luksa/fortune:env
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
    - name: config
      mountPath: /etc/nginx/conf.d/
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      #选择包含在卷中的条目
      items:
      #下列key对应的条目被包含
      - key: my-nginx-config.conf
        #条目的值存储在该文件中
        path: gzip.conf
```
path: gzip.conf表达条目的值存储在gzip.conf中

```
k exec fortune-configmap-volume-with-items -it -c web-server -- cat /etc/nginx/conf.d/gzip.conf
# server其实是内嵌到http块中的，协议级别配置
# 服务级别配置，一个http中可以有多个server。
server {
    # 监听端口
    listen              80;
    # 监听地址
    server_name         www.kubia-example.com;

    # 开启gzip压缩
    gzip on;

    # 设置压缩类型
    gzip_types text/plain application/xml;

    # 请求级别配置
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

**挂载某一文件夹会隐藏该文件夹中已存在的文件**
刚才的例子，我们将path: gzip.conf挂载到/etc/nginx/conf.d/下面，本质上gzip.conf会替换原来/etc/nginx/conf.d/下的所有文件。Linux系统挂载文件至非空文件夹的表现都是这样。所以在挂载文件到/etc这样重要的文件夹下面，要特别注意这个问题

**configMap独立条目作为文件被挂载且不隐藏文件夹中的其他文件**
使用subPath可以解决上面提及的问题
```
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: config
      mountPath: /etc/someconfig.conf
      subPath: myconfig.conf
```

**为configMap卷中的文件设置权限**
fortune-pod-configmap-volume-defaultMode.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
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
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: config
      mountPath: /tmp/whole-fortune-config-volume
      readOnly: true
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      #为fortune-config设置权限
      defaultMode: 493
```
这里需要注意,因为json中不能表示8进制，其实这个493对应的8进制的755，那么755对应的linux权限为:
lrwxr-xr-x.

接下来我们查看my-nginx-config.conf文件的权限
```
k exec fortune-configmap-volume -it -c web-server -- /bin/sh
/ # cd /etc/nginx/conf.d
/etc/nginx/conf.d # ls -l
total 0
lrwxrwxrwx    1 root     root            27 Dec  9 07:41 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root            21 Dec  9 07:41 sleep-interval -> ..data/sleep-interval
/etc/nginx/conf.d # cd /etc/nginx/conf.d/..data
/etc/nginx/conf.d/..2021_12_09_07_41_17.465373878 # pwd
/etc/nginx/conf.d/..data
/etc/nginx/conf.d/..2021_12_09_07_41_17.465373878 # ls -l
total 8
-rwxr-xr-x    1 root     root           484 Dec  9 07:41 my-nginx-config.conf
-rwxr-xr-x    1 root     root             3 Dec  9 07:41 sleep-interval
```

### 7.4.7 更新应用配合且不重启应用程序















