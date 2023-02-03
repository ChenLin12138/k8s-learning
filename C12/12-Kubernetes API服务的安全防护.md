# 12.Kubernetes API服务的安全防护
## 12.1 了解认证机制
### 12.1.2 ServiceAccount介绍
**了解ServiceAccount资源**
```
k get sa
NAME      SECRETS   AGE
default   1         11d
```
当前命名空间包含了一个default的ServiceAccount，其他额外的ServiceAccount需要添加。POD只能使用同样命名空间的ServiceAccount。
### 12.1.3 创建ServiceAccount
**创建ServiceAccount**
```
k create serviceaccount foo
```
查看ServiceAccount
```
k describe sa foo
Name:                foo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   foo-token-9s2d9
Tokens:              foo-token-9s2d9
Events:              <none>
```
查看自定义的ServiceAccount秘钥
```
k describe secret foo-token-9s2d9
Name:         foo-token-9s2d9
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: foo
              kubernetes.io/service-account.uid: b952de14-557d-45ee-bcf1-8b3c359da0da

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1111 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlNhQnYyUVAzWkFfMU5MLXpvM1JDZjQ2OTZtdFFleEp1Q0dSeUgzUXFkVVUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImZvby10b2tlbi05czJkOSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJmb28iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiOTUyZGUxNC01NTdkLTQ1ZWUtYmNmMS04YjNjMzU5ZGEwZGEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpmb28ifQ.gBR95ofCG_QaLUIhk7Az_8nwDeU2TI1v_FWfx7B3GfRa688JnEwpon1iMJNwrJGg6-38sMKp7LRCuqHw23z7s4ppVH2DRVg_6I8on2Sj_lTFHgDfiojb2z_b6MIfq24YStBSUxLlvZg_KEwQjFk4UQpsaKETNv_Xu5_UHkT4WUZtXQrBqBlyooLB4G5xgCT67lffrkz_Nw0bTJgNDlt_3m9uUSXMb0m51O9zcFRdxbNwA6QOju98x97F2QuDu7Z86e4-p3hry3zOZW2fQLiFcfV9FTe1uGVLYR_Hkned_XZ0sGP9O9grI6ja5a6VcjuByEi_SAHsNGugCi_TStGmQw
```
## 12.2 通过基于角色的权限控制加强集群安全
### 12.2.1 介绍RBAC授权插件
授权插件可以在Http方法级别上进行限制,例如http,GET,POST,PUT,PATCH,DELETE
### 12.2.2 介绍RBAC资源
RBAC授权规则通过4种资源进行配置
- Role（角色),Cluster Role(集群角色)，他们指定可以执行那些动作
- RoleBinding(角色绑定)和ClusterRoleBinding(集群角色绑定)。他们定义了谁可以做操作。
角色和角色绑定工作在命名空间级别，集群角色和集群角色绑定工作在集群级别。
下面的测试我没有能在本地的minikube上完成
### 12.2.3 使用Role和RoleBinding
**创建角色**
service-reader.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```
```
k create -f service-reader.yaml -n foo
```
**绑定角色到ServiceAccount**
角色定义了哪些操作可以执行，但是没有指定谁可以执行这个操作。要做到这一点，我们必须将角色绑定到一个主体。这个主体它可是是一个user,一个ServiceAccount或者一个组。
创建一个叫test的rolebinding，他与一个叫service-reader的role关联。
```
k create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
```
使用下面命名查询rolebinding和role的引用关系
```
k get rolebinding test -n foo -o yaml
```
**在角色绑定中使用其它命名空间的ServiceAccount**
一个namespace中的角色是可以绑定到另外一个命名空间上的ServiceAccount的

### 12.2.4 使用ClusterRole和ClusterRoleBinding
刚才我们看到的Role和RoleBinding都是namespace级别上的东西。但是如果我们有一个集群管理员。或者说我们想管理非命名空间级别上的资源例如Node,PersistenVolume,Namespace这些资源。常规的角色不能对这些资源或非资源的URL授权。但是ClusterRole可以。
**允许访问集群级别资源**
创建一个名为pv-reader的 clusterrole
```
k create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
```

获取名为reader的clusterrole定义
```
k get clusterrole pv-reader -o yaml
```
创建一个名pv-test的clusterrolebinding，他绑定的是clusterrole=pv-reader
**注意**这里使用的是一个ClusterRolebinding而不是RoleBinding.因为Rolbinding是不能给集群级别的资源做绑定的
```
k create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
```
**允许访问非资源类型的URL**
**总结Role,ClusterRole,Rolebinding,ClusterRoleBinding组合**

| 访问资源 |使用的角色类型  | 使用的绑定类型 |
| --- | --- | --- |
| 集群级别的资源(Nodes,PersistentVolumes) | ClusterRole | ClusterRoleBinding |
| 非资源URL(/api,/healthz) | ClusterRole| ClusterRoleBinding |
| 在任何命名空间中的资源（和跨所有命名空间的资源） | ClusterRole | ClusterRoleBinding |
| 在具体命名空间中的资源(在多个命名空间中重用这个相同的ClusterRole) |ClusterRole  | RoleBinding |
| 在具体命名空间中的资源(Role必须在每个命名空间中定义好) | Role | RoleBinding |

### 12.2.5 了解默认的ClusterRole和ClusterRoleBinding
```
k get clusterrolebindings
NAME                                            ROLE                                                                               AGE
kubeadm:get-nodes                               ClusterRole/kubeadm:get-nodes                                                      3d8h
kubeadm:kubelet-bootstrap                       ClusterRole/system:node-bootstrapper                                               3d8h
kubeadm:node-autoapprove-bootstrap              ClusterRole/system:certificates.k8s.io:certificatesigningrequests:nodeclient       3d8h
kubeadm:node-autoapprove-certificate-rotation   ClusterRole/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   3d8h
kubeadm:node-proxier                            ClusterRole/system:node-proxier                                                    3d8h
minikube-rbac                                   ClusterRole/cluster-admin                                                          3d8h
storage-provisioner                             ClusterRole/system:persistent-volume-provisioner                                   3d8h
system:coredns                                  ClusterRole/system:coredns                                                         3d8h
```