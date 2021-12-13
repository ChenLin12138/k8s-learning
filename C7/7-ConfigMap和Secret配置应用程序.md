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
使用环境变量或者命令行参数作为配置源的弊端在于无法在进程运行的时候进行更新。将ConfigMap暴露为卷则可以达到配置热更新的效果。无需重新创建pod或者启动容器。

先查看以前的配置，这里使用的是gzip on
```
kubectl exec -it fortune-configmap-volume -c web-server -- cat /etc/nginx/conf.d/my-nginx-config.conf
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
使用下面命令编辑configmap，完成gzip on 修改为gzip off
```
kubectl edit configmap fortune-config
configmap/fortune-config edited
```
再次查看发现并没有生效。
```
kubectl exec -it fortune-configmap-volume -c web-server -- cat /etc/nginx/conf.d/my-nginx-config.conf
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
这里需要reload一下nginx
```
kubectl exec -it fortune-configmap-volume -c web-server -- nginx -s reload
2021/12/13 03:17:27 [notice] 97#97: signal process started
```
再次查看，发现生效，现在修改为gzip off
···
kubectl exec -it fortune-configmap-volume -c web-server -- cat /etc/nginx/conf.d/my-nginx-config.conf
server {
    listen              80;
    server_name         www.kubia-example.com;

    gzip off;
    gzip_types text/plain application/xml;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

}
···

**了解文件被自动更新的过程**
可以看到他这边实际上做了一个junction的东西。之前的配置都在..2021_12_13_03_17_00.880589937这个软连接中。现在替换为..2021_12_13_03_17_00.880589937
```
kubectl exec -it fortune-configmap-volume -c web-server -- ls -lA /etc/nginx/conf.d
total 4
drwxr-xr-x    2 root     root          4096 Dec 13 03:17 ..2021_12_13_03_17_00.880589937
lrwxrwxrwx    1 root     root            31 Dec 13 03:17 ..data -> ..2021_12_13_03_17_00.880589937
lrwxrwxrwx    1 root     root            27 Dec  3 09:11 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root            21 Dec  3 09:11 sleep-interval -> ..data/sleep-interval

```
每次更新后会重新挂载新的软连接
```
kubectl exec -it fortune-configmap-volume -c web-server -- ls -lA /etc/nginx/conf.d
total 4
drwxr-xr-x    2 root     root          4096 Dec 13 05:26 ..2021_12_13_05_26_22.797698573
lrwxrwxrwx    1 root     root            31 Dec 13 05:26 ..data -> ..2021_12_13_05_26_22.797698573
lrwxrwxrwx    1 root     root            27 Dec  3 09:11 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root            21 Dec  3 09:11 sleep-interval -> ..data/sleep-interval
```

**挂载至已存在文件夹的文件不会被更新**
如果挂载的文件是容器中的单个文件而不是整个卷，ConfigMap更新之后对应的文件不会被更新。如果想修改单个文件同时自动修改这个文件，一种方案是挂载完整卷到不同的文件夹并创建指向所需文件的符号连接。
符号链接可以原生创建在容器镜像中，可以在容器启动时创建。

## 7.5 使用Secret给容器传递敏感数据
现在我们来尝试传递一些敏感数据

### 7.5.1 
Secret是一种敏感资源。使用方式和ConfigMap非常类似。
- 将Secret条目作为环境变量传递给容器
- 将Secret条目暴露为卷中的文件
k8s通过仅仅将Secret分发到需要访问的Secret的pod所在的机器节点来保证安全性。另外Secret只会存储在节点的内存中，永不会写入物理存储。这样从节点上删除Secret时就不需要擦除磁盘了。

### 7.5.2 默认令牌Secret介绍
查询secrets
```
k get secrets
NAME                  TYPE                                  DATA   AGE
default-token-t5vzk   kubernetes.io/service-account-token   3      70d
```

查询secrets内容
可以看到secrets包含3个条目
ca.cet
namespaces
token
```
k describe secrets
Name:         default-token-t5vzk
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 3784ea6e-f301-42d8-a6f9-fdd5a1f58ffc
Type:  kubernetes.io/service-account-token
Data
====
ca.crt:     1111 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkZ1WUNldE44NzlXR0J5VlN5eVdaLVhPaWJ3RXdlU3BDN2dQa3FVTzEtVHMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tdDV2emsiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjM3ODRlYTZlLWYzMDEtNDJkOC1hNmY5LWZkZDVhMWY1OGZmYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.XcNBwx3LXxK4ohfkkILXo6zryWd8F4hLzDIcXYvIM8Khp-_fQ9l9cvSfiVxgLA07qGHnchPMQcJWpaigh3qKtTIGIODNC_uZWOBFF-lwulbvFKmMUzChzCqdlW8zbehigqJaHsqulmxPj4VgyeH89S2Zfxf0T21XgFSIJznVd2D2EPjH-DD0Mo-hKmgro2mSi8gCJqtX251_a1F9ivNbukkXEcveelyFRO28Ej0WpxsGTI8YwA23jlmi9kI3pOmjPK2dOYfmz-ye719bViis_lsRIlfw_TbaVkshPaWQRGFgqLEXQRh549sPu60PZxtNsfOrNSJvRSYZ1x4WEYm0aw
```
可以发现下面每一个container中都使用到了secret
```
k describe pod
Name:         fortune-configmap-volume
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Fri, 03 Dec 2021 17:11:14 +0800
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           172.17.0.4
IPs:
  IP:  172.17.0.4
Containers:
  html-generator:
    Container ID:   docker://1c652dd2cbd14c8d7faed6227ae1aaa2efe2c9fad2e7ca0b501fc3fdfbbdf106
    Image:          luksa/fortune:env
    Image ID:       docker-pullable://luksa/fortune@sha256:8af10b8eb1b1dcc6512e0061c0722db43f4795aefcde43b524b1c8b302611dde
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:36 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:11:15 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  1
    Environment:
      INTERVAL:  <set to the key 'sleep-interval' of config map 'fortune-config'>  Optional: false
    Mounts:
      /var/htdocs from html (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-dbjsj (ro)
  web-server:
    Container ID:   docker://ef4c6d675e888e6060fa114cac89da2e7e37913cf5266a0f857b7594db6fbe30
    Image:          nginx:alpine
    Image ID:       docker-pullable://nginx@sha256:686aac2769fd6e7bab67663fd38750c135b72d993d0bb0a942ab02ef647fc9c3
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:36 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:11:15 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /etc/nginx/conf.d from config (ro)
      /tmp/whole-fortune-config-volume from config (ro)
      /usr/share/nginx/html from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-dbjsj (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  html:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      fortune-config
    Optional:  false
  kube-api-access-dbjsj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>


Name:         fortune-env
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Fri, 03 Dec 2021 17:20:10 +0800
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           172.17.0.7
IPs:
  IP:  172.17.0.7
Containers:
  html-generator:
    Container ID:   docker://bb1312062a81c9971fd8dc3ec21d5ea009589176e26e0d2efceccab69b17b934
    Image:          luksa/fortune:env
    Image ID:       docker-pullable://luksa/fortune@sha256:8af10b8eb1b1dcc6512e0061c0722db43f4795aefcde43b524b1c8b302611dde
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:36 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:20:10 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  1
    Environment:
      INTERVAL:  30
    Mounts:
      /var/htdocs from html (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kqg8g (ro)
  web-server:
    Container ID:   docker://9b008fa046feea56840d756266ba37b2662eb9e8b3ac405b74882a23da96156e
    Image:          nginx:alpine
    Image ID:       docker-pullable://nginx@sha256:686aac2769fd6e7bab67663fd38750c135b72d993d0bb0a942ab02ef647fc9c3
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:36 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:20:10 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  1
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kqg8g (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  html:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
  kube-api-access-kqg8g:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>


Name:         fortune-env-from-configmap
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Fri, 26 Nov 2021 16:59:41 +0800
Labels:       <none>
Annotations:  <none>
Status:       Running
IP:           172.17.0.3
IPs:
  IP:  172.17.0.3
Containers:
  html-generator:
    Container ID:   docker://25f172e37389dbd6d0280e8d64856a8d1374a671d37cdde44ffe7e8caa5d2484
    Image:          luksa/fortune:env
    Image ID:       docker-pullable://luksa/fortune@sha256:8af10b8eb1b1dcc6512e0061c0722db43f4795aefcde43b524b1c8b302611dde
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:35 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:01:27 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  5
    Environment:
      INTERVAL:  <set to the key 'sleep-interval' of config map 'fortune-config'>  Optional: false
    Mounts:
      /var/htdocs from html (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ghj52 (ro)
  web-server:
    Container ID:   docker://d2b837e1b06362494cedb6d57cf80b5aa6f06417a669f28574d0d717a92121ec
    Image:          library/nginx:alpine
    Image ID:       docker-pullable://nginx@sha256:686aac2769fd6e7bab67663fd38750c135b72d993d0bb0a942ab02ef647fc9c3
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:35 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:01:27 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  5
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from html (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-ghj52 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  html:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  kube-api-access-ghj52:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```
我们接下来取一个Container来观察，我们用html-generator作为例子
```
  html-generator:
    Container ID:   docker://1c652dd2cbd14c8d7faed6227ae1aaa2efe2c9fad2e7ca0b501fc3fdfbbdf106
    Image:          luksa/fortune:env
    Image ID:       docker-pullable://luksa/fortune@sha256:8af10b8eb1b1dcc6512e0061c0722db43f4795aefcde43b524b1c8b302611dde
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 13 Dec 2021 10:51:36 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Fri, 03 Dec 2021 17:11:15 +0800
      Finished:     Mon, 13 Dec 2021 10:50:57 +0800
    Ready:          True
    Restart Count:  1
    Environment:
      INTERVAL:  <set to the key 'sleep-interval' of config map 'fortune-config'>  Optional: false
    Mounts:
      /var/htdocs from html (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-dbjsj (ro)
```

```
k exec -it fortune-env-from-configmap -c html-generator -- ls -l /var/run/secrets/kubernetes.io/serviceaccount
total 0
lrwxrwxrwx 1 root root 13 Dec 13 02:51 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Dec 13 02:51 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Dec 13 02:51 token -> ..data/token
```

### 7.5.3 创建Secret
现在我们想创建自己地小型Secret，改进fortune-serving的Nginx容器的配置。使得它能服务于https色流量，我们创建私钥和证书，由于需要保证私钥的安全性，可以将其与证书同时存入Secret
首先在本地机器上生成证书与私钥文件，当然也可以世界使用本书代码文档中的相应文件。
创建一个generic类型的secret，来自于文件夹fortune-https
```
k create secret generic fortune-https --from-file=fortune-https
secret/fortune-https created
```
### 7.5.4 对比ConfigMap与Secret
Secret与ConfigMap有比较大的区别。
```
k get secret fortune-https -o yaml
C:\NotBackedUp\learn\k8s-learning-master\C7>k get secret fortune-https -o yaml
apiVersion: v1
data:
  foo: YmFy
  https.cert: CgoKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIiBkYXRhLWNvbG9yLW1vZGU9ImF1dG8iIGRhdGEtbGlnaHQtdGhlbWU9ImxpZ2h0IiBkYXRhLWRhcmstdGhlbWU9ImRhcmsiPgogIDxoZWFkPgogICAgPG1ldGEgY2hhcnNldD0idXRmLTgiPgogIDxsaW5rIHJlbD0iZG5zLXByZWZldGNoIiBocmVmPSJodHRwczovL2dpdGh1Yi5naXRodWJhc3NldHMuY29tIj4KICA8bGluayByZWw9ImRucy1wcmVmZXRjaCIgaHJlZj0iaHR0cHM6Ly9hdmF0YXJzLmdpdGh1YnVzZXJjb250ZW50LmNvbSI+CiAgPGxpbmsgcmVsPSJkbnMtcHJlZmV0Y2giIGhyZWY9Imh0dHBzOi8vZ2l0aHViLWNsb3VkLnMzLmFtYXpvbmF3cy5jb20iPgogIDxsaW5rIHJlb
```
可以看到的是secret条目内容以Bash64格式编码，而ConfigMap直接以纯文本展示。
**以二进制数据创建Secret**

### 7.5.5 在pod中使用Secret
fortune-https Sercet已经包含了证书与密钥文件，接下来需要做的是配置Nginx服务器去使用它。

**修改fortune-config ConfigMap以开启Https**
创建一个名为configmap-ssl的文件夹
将sleep-interval和ssl.conf放入其中
fortune-config/my-nginx-config.conf
```
server {
    listen              80;
    listen              443 ssl;
    server_name         www.kubia-example.com;
    ssl_certificate     certs/https.cert;
    ssl_certificate_key certs/https.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```
重新创建带ssl的
```
k create configmap fortune-config --from-file=configmap-ssl
configmap/fortune-config created
``