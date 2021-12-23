# 8. 从应用访问pod元数据以及其他资源
## 8.1 通过Downward API传递元数据
configmap和secret卷存储pod调度和运行前预设的数据。那么不能预设的数据，例如pod ip, pod name则通过Downward API保留。
### 8.1.1 了解可用的元数据
- pod的名称
- pod的ip
- pod所在的命名空间
- pod运行节点的名称
- pod运行所归属的服务账号名称
- 每个容器请求的cpu和内存的使用量
- 每个容器可以使用的cpu和内存的限制
- pod的标签
- pod的注解

### 8.1.2 通过环境变量暴露元数据

根据下面内容配置一个名为downward的pod
downward-api-env.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 6Mi
    #通过环境变量的方式，暴露元数据
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

查看环境变量
所有这个容器中运行的进程都可以读取并使用它们需要的变量

```
k exec downward  -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=downward
SERVICE_ACCOUNT=default
CONTAINER_CPU_REQUEST_MILLICORES=15
CONTAINER_MEMORY_LIMIT_KIBIBYTES=6144
POD_NAME=downward
POD_NAMESPACE=default
POD_IP=172.17.0.9
NODE_NAME=minikube
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
HOME=/root
```
### 8.1.3 通过downwardAPI卷来传递元数据
downward-api-volume.yaml
下面是通过一个卷来传递元素据，上面的例子是通过环境变量来传递元素据。这一章里面讲的东西刚好和之前是反的。这里downward要实现的内容是通过
env或者卷的方式，暴露pod的运行参数。
```
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 6Mi
    volumeMounts:
    - name: downward
      mountPath: /etc/downward
  #通过卷的方式，暴露元数据
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "podName"
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
#resourceFieldRef必须指定容器的名称
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```
查看这些卷
```
k exec downward -- ls -lL /etc/downward
total 24
-rw-r--r--    1 root     root           134 Dec 15 08:23 annotations
-rw-r--r--    1 root     root             2 Dec 15 08:23 containerCpuRequestMilliCores
-rw-r--r--    1 root     root             7 Dec 15 08:23 containerMemoryLimitBytes
-rw-r--r--    1 root     root             9 Dec 15 08:23 labels
-rw-r--r--    1 root     root             8 Dec 15 08:23 podName
-rw-r--r--    1 root     root             7 Dec 15 08:23 podNamespace
```

**修改标签和注解**
可以在pod运行时修改标签和注解。当标签和注解被修改后，k8s会更新有关相关信息的文件，从而使pod可以获取最新的数据。而如果通过环境变量
的方式暴露标签和注解，在环境变量方式下，一旦标签和注解被修改了，新的值将无法暴露。

**在卷的定义中引用容器级的元数据**
- 在暴露容器级的元数据时，必须指定引用资源对应的容器名称，因为volumes的定义是pod级别的，不是容器级别的。
```
spec:
  volumes:
  - name: downward
  downwardAPI:
    items:
    - path: "containerCpuRequestMilliCores"
      resourceFieldRef:
      containerName: main  // 必须指定容器名称
      resource: requests.cpu
      divisor: 1m
```

**合适使用Downward API**
Downward API的方式很简单的暴露元素据，但是暴露的数据是有限的。我们可以使用Kubernetes API服务的方式来暴露更的元数据

## 8.2 与Kubernetes API服务器交互

### 8.2.1 探究Kubernetes REST API 
获取服务器的url,服务器跑在https://127.0.0.1:51626上面
```
k cluster-info
Kubernetes control plane is running at https://127.0.0.1:51626
CoreDNS is running at https://127.0.0.1:51626/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

我们想直接交互，但是没有成功
```
$ curl https://127.0.0.1:51626 -k
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   233  100   233    0     0   7766      0 --:--:-- --:--:-- --:--:--  8629{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

**通过kubectl proxy访问API服务器**
kubectl proxy命令启动一个代理服务器来接受本机的http连接，并转发至API服务器
```
k proxy
Starting to serve on 127.0.0.1:8001
```
通过代理服务器返回的kubernetes api的json信息
```
$ curl localhost:8001
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1",
    "/apis/events.k8s.io/v1beta1",
    "/apis/flowcontrol.apiserver.k8s.io",
    "/apis/flowcontrol.apiserver.k8s.io/v1beta1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/aggregator-reload-proxy-client-cert",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/priority-and-fairness-config-consumer",
    "/healthz/poststarthook/priority-and-fairness-config-producer",
    "/healthz/poststarthook/priority-and-fairness-filter",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/aggregator-reload-proxy-client-cert",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/priority-and-fairness-config-consumer",
    "/livez/poststarthook/priority-and-fairness-config-producer",
    "/livez/poststarthook/priority-and-fairness-filter",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
100  5448    0  5448    0     0   147k      0 --:--:-- --:--:-- --:--:--  147k,
    "/openapi/v2",
    "/openid/v1/jwks",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/informer-sync",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/aggregator-reload-proxy-client-cert",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/priority-and-fairness-config-consumer",
    "/readyz/poststarthook/priority-and-fairness-config-producer",
    "/readyz/poststarthook/priority-and-fairness-filter",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```

**通过kubectl proxy研究kubernetes API**

**研究批量API组的REST endpoint**
从这里可以看出来我们有batch/v1和batch/v1beta1两个版本，但是preferredVersion的版本是batch/v1
```
$ curl http://localhost:8001/apis/batch
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   314  100   314    0     0  11214      0 --:--:-- --:--:-- --:--:-- 11214{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "batch",
  "versions": [
    {
      "groupVersion": "batch/v1",
      "version": "v1"
    },
    {
      "groupVersion": "batch/v1beta1",
      "version": "v1beta1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "batch/v1",
    "version": "v1"
  }
}
```

那么我们接下来调查batch/v1这个版本
下面的数据可以从resources中找出资源的类型cronjobs，它支持什么样的操作create，delete
```
$ curl http://localhost:8001/apis/batch/v1
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1278  100  1278    0     0  53250      0 --:--:-- --:--:-- --:--:-- 53250{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "batch/v1",
  "resources": [
    {
      "name": "cronjobs",
      "singularName": "",
      "namespaced": true,
      "kind": "CronJob",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "cj"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "sd5LIXh4Fjs="
    },
    {
      "name": "cronjobs/status",
      "singularName": "",
      "namespaced": true,
      "kind": "CronJob",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    },
    {
      "name": "jobs",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "mudhfqk/qZY="
    },
    {
      "name": "jobs/status",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

**列举集群中所有的Job实例**
你自己启动的job资源会在items中
```
$ curl http://localhost:8001/apis/batch/v1/jobs
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "resourceVersion": "1250448"
  },
  "items": [
    {
      "metadata": {
        "name": "ingress-nginx-admission-create",
        "namespace": "ingress-nginx",
        "uid": "aaf068a1-c688-40f6-bbab-6eb3e9ae2e92",
        "resourceVersion": "561",
        "generation": 1,
        "creationTimestamp": "2021-10-04T02:56:24Z",
        "labels": {
          "app.kubernetes.io/component": "admission-webhook",
          "app.kubernetes.io/instance": "ingress-nginx",
          "app.kubernetes.io/name": "ingress-nginx"
        },
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"batch/v1\",\"kind\":\"Job\",\"metadata\":{\"annotations\":{},\"labels\":{\"app.kubernetes.io/component\":\"admission-webhook\",\"app.kubernetes.io/instance\":\"ingress-nginx\",\"app.kubernetes.io/name\":\"ingress-nginx\"},\"name\":\"ingress-nginx-admission-create\",\"namespace\":\"ingress-nginx\"},\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"app.kubernetes.io/component\":\"admission-webhook\",\"app.kubernetes.io/instance\":\"ingress-nginx\",\"app.kubernetes.io/name\":\"ingress-nginx\"},\"name\":\"ingress-nginx-admission-create\"},\"spec\":{\"containers\":[{\"args\":[\"create\",\"--host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc\",\"--namespace=$(POD_NAMESPACE)\",\"--secret-name=ingress-nginx-admission\"],\"env\":[{\"name\":\"POD_NAMESPACE\",\"valueFrom\":{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"}}}],\"image\":\"k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0@sha256:f3b6b39a6062328c095337b4cadcefd1612348fdd5190b1dcbcb9b9e90bd8068\",\"imagePullPolicy\":\"IfNotPresent\",\"name\":\"create\"}],\"restartPolicy\":\"OnFailure\",\"securityContext\":{\"runAsNonRoot\":true,\"runAsUser\":2000},\"serviceAccountName\":\"ingress-nginx-admission\"}}}}\n"
        },
        "managedFields": [
          {
            "manager": "kube-controller-manager",
            "operation": "Update",
            "apiVersion": "batch/v1",
            "time": "2021-10-04T02:56:24Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:status":{"f:active":{},"f:startTime":{}}},
            "subresource": "status"
          },
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "batch/v1",
            "time": "2021-10-04T02:56:24Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app.kubernetes.io/component":{},"f:app.kubernetes.io/instance":{},"f:app.kubernetes.io/name":{}}},"f:spec":{"f:backoffLimit":{},"f:completionMode":{},"f:completions":{},"f:parallelism":{},"f:suspend":{},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app.kubernetes.io/component":{},"f:app.kubernetes.io/instance":{},"f:app.kubernetes.io/name":{}},"f:name":{}},"f:spec":{"f:containers":{"k:{\"name\":\"create\"}":{".":{},"f:args":{},"f:env":{".":{},"k:{\"name\":\"POD_NAMESPACE\"}":{".":{},"f:name":{},"f:valueFrom":{".":{},"f:fieldRef":{}}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{".":{},"f:runAsNonRoot":{},"f:runAsUser":{}},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{}}}}}
          }
        ]
      },
      "spec": {
        "parallelism": 1,
        "completions": 1,
        "backoffLimit": 6,
        "selector": {
          "matchLabels": {
            "controller-uid": "aaf068a1-c688-40f6-bbab-6eb3e9ae2e92"
          }
        },
        "template": {
          "metadata": {
            "name": "ingress-nginx-admission-create",
            "creationTimestamp": null,
            "labels": {
              "app.kubernetes.io/component": "admission-webhook",
              "app.kubernetes.io/instance": "ingress-nginx",
              "app.kubernetes.io/name": "ingress-nginx",
              "controller-uid": "aaf068a1-c688-40f6-bbab-6eb3e9ae2e92",
              "job-name": "ingress-nginx-admission-create"
            }
          },
          "spec": {
            "containers": [
              {
                "name": "create",
                "image": "k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0@sha256:f3b6b39a6062328c095337b4cadcefd1612348fdd5190b1dcbcb9b9e90bd8068",
                "args": [
                  "create",
                  "--host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc",
                  "--namespace=$(POD_NAMESPACE)",
                  "--secret-name=ingress-nginx-admission"
                ],
                "env": [
                  {
                    "name": "POD_NAMESPACE",
                    "valueFrom": {
                      "fieldRef": {
                        "apiVersion": "v1",
                        "fieldPath": "metadata.namespace"
                      }
                    }
                  }
                ],
                "resources": {

                },
                "terminationMessagePath": "/dev/termination-log",
                "terminationMessagePolicy": "File",
                "imagePullPolicy": "IfNotPresent"
              }
            ],
            "restartPolicy": "OnFailure",
            "terminationGracePeriodSeconds": 30,
            "dnsPolicy": "ClusterFirst",
            "serviceAccountName": "ingress-nginx-admission",
            "serviceAccount": "ingress-nginx-admission",
            "securityContext": {
              "runAsUser": 2000,
              "runAsNonRoot": true
            },
            "schedulerName": "default-scheduler"
          }
        },
        "completionMode": "NonIndexed",
        "suspend": false
      },
      "status": {
        "startTime": "2021-10-04T02:56:24Z",
        "active": 1
      }
    },
    {
      "metadata": {
        "name": "ingress-nginx-admission-patch",
        "namespace": "ingress-nginx",
        "uid": "e573864e-c90e-424f-b18f-13e965b14ca5",
        "resourceVersion": "565",
        "generation": 1,
        "creationTimestamp": "2021-10-04T02:56:24Z",
        "labels": {
          "app.kubernetes.io/component": "admission-webhook",
          "app.kubernetes.io/instance": "ingress-nginx",
          "app.kubernetes.io/name": "ingress-nginx"
        },
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"batch/v1\",\"kind\":\"Job\",\"metadata\":{\"annotations\":{},\"labels\":{\"app.kubernetes.io/component\":\"admission-webhook\",\"app.kubernetes.io/instance\":\"ingress-nginx\",\"app.kubernetes.io/name\":\"ingress-nginx\"},\"name\":\"ingress-nginx-admission-patch\",\"namespace\":\"ingress-nginx\"},\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"app.kubernetes.io/component\":\"admission-webhook\",\"app.kubernetes.io/instance\":\"ingress-nginx\",\"app.kubernetes.io/name\":\"ingress-nginx\"},\"name\":\"ingress-nginx-admission-patch\"},\"spec\":{\"containers\":[{\"args\":[\"patch\",\"--webhook-name=ingress-nginx-admission\",\"--namespace=$(POD_NAMESPACE)\",\"--patch-mutating=false\",\"--secret-name=ingress-nginx-admission\",\"--patch-failure-policy=Fail\"],\"env\":[{\"name\":\"POD_NAMESPACE\",\"valueFrom\":{\"fieldRef\":{\"fieldPath\":\"metadata.namespace\"}}}],\"image\":\"k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0@sha256:f3b6b39a6062328c095337b4cadcefd1612348fdd5190b1dcbcb9b9e90bd8068\",\"imagePullPolicy\":\"IfNotPresent\",\"name\":\"patch\"}],\"restartPolicy\":\"OnFailure\",\"securityContext\":{\"runAsNonRoot\":true,\"runAsUser\":2000},\"serviceAccountName\":\"ingress-nginx-admission\"}}}}\n"
        },
        "managedFields": [
          {
            "manager": "kube-controller-manager",
            "operation": "Update",
            "apiVersion": "batch/v1",
            "time": "2021-10-04T02:56:24Z",
            "fieldsType": "FieldsV1",
100 12116    0 12116    0     0   348k      0 --:--:-- --:--:-- --:--:--  348k"f:status":{"f:active":{},"f:startTime":{}}},
            "subresource": "status"
          },
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "batch/v1",
            "time": "2021-10-04T02:56:24Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {"f:metadata":{"f:annotations":{".":{},"f:kubectl.kubernetes.io/last-applied-configuration":{}},"f:labels":{".":{},"f:app.kubernetes.io/component":{},"f:app.kubernetes.io/instance":{},"f:app.kubernetes.io/name":{}}},"f:spec":{"f:backoffLimit":{},"f:completionMode":{},"f:completions":{},"f:parallelism":{},"f:suspend":{},"f:template":{"f:metadata":{"f:labels":{".":{},"f:app.kubernetes.io/component":{},"f:app.kubernetes.io/instance":{},"f:app.kubernetes.io/name":{}},"f:name":{}},"f:spec":{"f:containers":{"k:{\"name\":\"patch\"}":{".":{},"f:args":{},"f:env":{".":{},"k:{\"name\":\"POD_NAMESPACE\"}":{".":{},"f:name":{},"f:valueFrom":{".":{},"f:fieldRef":{}}}},"f:image":{},"f:imagePullPolicy":{},"f:name":{},"f:resources":{},"f:terminationMessagePath":{},"f:terminationMessagePolicy":{}}},"f:dnsPolicy":{},"f:restartPolicy":{},"f:schedulerName":{},"f:securityContext":{".":{},"f:runAsNonRoot":{},"f:runAsUser":{}},"f:serviceAccount":{},"f:serviceAccountName":{},"f:terminationGracePeriodSeconds":{}}}}}
          }
        ]
      },
      "spec": {
        "parallelism": 1,
        "completions": 1,
        "backoffLimit": 6,
        "selector": {
          "matchLabels": {
            "controller-uid": "e573864e-c90e-424f-b18f-13e965b14ca5"
          }
        },
        "template": {
          "metadata": {
            "name": "ingress-nginx-admission-patch",
            "creationTimestamp": null,
            "labels": {
              "app.kubernetes.io/component": "admission-webhook",
              "app.kubernetes.io/instance": "ingress-nginx",
              "app.kubernetes.io/name": "ingress-nginx",
              "controller-uid": "e573864e-c90e-424f-b18f-13e965b14ca5",
              "job-name": "ingress-nginx-admission-patch"
            }
          },
          "spec": {
            "containers": [
              {
                "name": "patch",
                "image": "k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0@sha256:f3b6b39a6062328c095337b4cadcefd1612348fdd5190b1dcbcb9b9e90bd8068",
                "args": [
                  "patch",
                  "--webhook-name=ingress-nginx-admission",
                  "--namespace=$(POD_NAMESPACE)",
                  "--patch-mutating=false",
                  "--secret-name=ingress-nginx-admission",
                  "--patch-failure-policy=Fail"
                ],
                "env": [
                  {
                    "name": "POD_NAMESPACE",
                    "valueFrom": {
                      "fieldRef": {
                        "apiVersion": "v1",
                        "fieldPath": "metadata.namespace"
                      }
                    }
                  }
                ],
                "resources": {

                },
                "terminationMessagePath": "/dev/termination-log",
                "terminationMessagePolicy": "File",
                "imagePullPolicy": "IfNotPresent"
              }
            ],
            "restartPolicy": "OnFailure",
            "terminationGracePeriodSeconds": 30,
            "dnsPolicy": "ClusterFirst",
            "serviceAccountName": "ingress-nginx-admission",
            "serviceAccount": "ingress-nginx-admission",
            "securityContext": {
              "runAsUser": 2000,
              "runAsNonRoot": true
            },
            "schedulerName": "default-scheduler"
          }
        },
        "completionMode": "NonIndexed",
        "suspend": false
      },
      "status": {
        "startTime": "2021-10-04T02:56:24Z",
        "active": 1
      }
    }
  ]
}
```

**通过名称恢复一个指定的Job实例**
没查到对应的资源
```
$ curl http://localhost:8001/apis/batch/v1/namespaces/default/jobs/ingress-nginx-admission-create
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   311  100   311    0     0   8638      0 --:--:-- --:--:-- --:--:--  8638{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "jobs.batch \"ingress-nginx-admission-create\" not found",
  "reason": "NotFound",
  "details": {
    "name": "ingress-nginx-admission-create",
    "group": "batch",
    "kind": "jobs"
  },
  "code": 404
}
```
### 8.2.2 从pod内部与API服务器进行交互
想要从Pod内部访问他需要确定下面三件事情。
- 确定API服务器的位置
- 确保是与API服务器进行交互，而不是一个冒名者
- 通过服务器的认证，否则将来不能查看任何内容以及进行任何操作

**运行一个pod来尝试与API服务器进行通信**
curl.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: main
    image: rancher/curl
    command: ["sleep", "9999999"]
```
我们要明白以下事情，kubernetess API也是通过service暴露出来的，那么我们看看现在有哪些service暴露出来了，他的IP地址是10.96.0.1:443
```
$ k get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   86d
```
容器中的env的KUBERNETES_SERVICE_HOST和KUBERNETES_SERVICE_PORT 和k8s API的是一致的
```
k exec -it curl -- /bin/sh
/ # env | grep KUBERNETES_SERVICE
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
```

**验证服务器身份**

```
k exec -it curl -- /bin/sh
/var/run/secrets/kubernetes.io/serviceaccount # ls -l
total 0
lrwxrwxrwx    1 root     root            13 Dec 22 07:23 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root            16 Dec 22 07:23 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            12 Dec 22 07:23 token -> ..data/token
```

通过添加--cacert参数访问https
```
/var/run/secrets/kubernetes.io/serviceaccount # curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
```

导入证书
```
/var/run/secrets/kubernetes.io/serviceaccount # export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

导入证书后，可以不使用--cacert来访问API服务器
```
/var/run/secrets/kubernetes.io/serviceaccount # curl https://kubernetes
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
```

**获取API服务器授权**
```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

为了解决403的问题，我们需要给所有的cluster-admin，这是关闭基于角色的访问控制(RBAL)
```
k create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts
clusterrolebinding.rbac.authorization.k8s.io/permissive-binding created
```

上面描述的问题cert解决ssl访问的问题，token解决授权的问题。
但是我没搞定。我放弃了。

**获取当前运行pod所在的命名空间**

由于上面的没搞定，命名空间也凉了。

### 8.2.3 通过ambassador容器进化与API服务器的交互
**通过ambassador容器简化与api服务的交互**
如果每一个容器与api交互都需要设置https的证书和token,那么事情变得非常的复杂。我们接下来想完成的事情是所有的应用容器均通过http的方式和ambassador容器交互，而ambassador容器与API服务器通过https交互。

**运行带有附加ambassador容器的curl pod**
```
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: rancher/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

然后你会发现，确实不需要证书和token,但是一样的还是错。
```
k  exec -it curl-with-ambassador -c main -- /bin/sh
/ # curl localhost:8001
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:default:default\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}/ #
```
加密，授权，传输，服务器验证工作交给ambassador容器中的kubectl proxy来做了

### 8.2.4 使用客户端库与API服务器交互
刚才我们体验的是使用ambassador容器kubectl-proxy的好处。如果我们的应用仅仅需要在API服务器中执行一些简单的操作，往往尅使用一个标准的客户端来执行一些简单的http操作请求。但是如果执行复杂的API请求来说，使用某个已有的kubernetes api客户端会更好一些。
**使用已有的客户端库**
**一个使用Fabric8 Java库与Kubernetes进行交互的例子**
一个Fabirc8 Kubernetes客户端李处一个Java应用中的服务。














