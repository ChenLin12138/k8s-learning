# 11.了解Kubernetes机理
## 11.1 了解架构
### 11.1.4 API服务器如何通知客户端资源变更
监听创建删除pod事件
```
k get pods --watch
NAME      READY   STATUS    RESTARTS      AGE
kubia-0   1/1     Running   1 (22m ago)   2d21h
kubia-1   1/1     Running   1 (22m ago)   2d23h
```
## 11.2 控制器如何协作
### 11.2.2 事件链
观察控制器发出的事件
```
k get events --watch
LAST SEEN   TYPE      REASON                    OBJECT                       MESSAGE
2d21h       Normal    Killing                   pod/kubia-0                  Stopping container kubia
2d21h       Normal    Scheduled                 pod/kubia-0                  Successfully assigned default/kubia-0 to minikube
2d21h       Warning   FailedKillPod             pod/kubia-0                  error killing pod: failed to "KillContainer" for "kubia" with KillContainerError: "rpc error: code = Unknown desc = Error response from daemon: No such container: b0e80e79246194fd21900eb8dbe96d497563cffa4057461b819bef6e620f8fb1"
2d21h       Normal    Pulling                   pod/kubia-0                  Pulling image "luksa/kubia-pet"
2d21h       Normal    Pulled                    pod/kubia-0                  Successfully pulled image "luksa/kubia-pet" in 2.903372891s
2d21h       Normal    Created                   pod/kubia-0                  Created container kubia
2d21h       Normal    Started                   pod/kubia-0                  Started container kubia
24m         Normal    SandboxChanged            pod/kubia-0                  Pod sandbox changed, it will be killed and re-created.
24m         Normal    Pulling                   pod/kubia-0                  Pulling image "luksa/kubia-pet"
24m         Normal    Pulled                    pod/kubia-0                  Successfully pulled image "luksa/kubia-pet" in 5.326408684s
24m         Normal    Created                   pod/kubia-0                  Created container kubia
24m         Normal    Started                   pod/kubia-0                  Started container kubia
24m         Normal    SandboxChanged            pod/kubia-1                  Pod sandbox changed, it will be killed and re-created.
24m         Normal    Pulling                   pod/kubia-1                  Pulling image "luksa/kubia-pet"
24m         Normal    Pulled                    pod/kubia-1                  Successfully pulled image "luksa/kubia-pet" in 2.901512226s
24m         Normal    Created                   pod/kubia-1                  Created container kubia
24m         Normal    Started                   pod/kubia-1                  Started container kubia
2d21h       Normal    Scheduled                 pod/kubia-bcf9bb974-5z9g8    Successfully assigned default/kubia-bcf9bb974-5z9g8 to minikube
2d21h       Normal    Pulled                    pod/kubia-bcf9bb974-5z9g8    Container image "luksa/kubia:v2" already present on machine
2d21h       Normal    Created                   pod/kubia-bcf9bb974-5z9g8    Created container nodejs
2d21h       Normal    Started                   pod/kubia-bcf9bb974-5z9g8    Started container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-5z9g8    Stopping container nodejs
2d21h       Normal    Scheduled                 pod/kubia-bcf9bb974-89b8z    Successfully assigned default/kubia-bcf9bb974-89b8z to minikube
2d21h       Normal    Pulled                    pod/kubia-bcf9bb974-89b8z    Container image "luksa/kubia:v2" already present on machine
2d21h       Normal    Created                   pod/kubia-bcf9bb974-89b8z    Created container nodejs
2d21h       Normal    Started                   pod/kubia-bcf9bb974-89b8z    Started container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-89b8z    Stopping container nodejs
2d21h       Normal    Scheduled                 pod/kubia-bcf9bb974-l9dkq    Successfully assigned default/kubia-bcf9bb974-l9dkq to minikube
2d21h       Normal    Pulled                    pod/kubia-bcf9bb974-l9dkq    Container image "luksa/kubia:v2" already present on machine
2d21h       Normal    Created                   pod/kubia-bcf9bb974-l9dkq    Created container nodejs
2d21h       Normal    Started                   pod/kubia-bcf9bb974-l9dkq    Started container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-l9dkq    Stopping container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-l9v9g    Stopping container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-srl6j    Stopping container nodejs
2d21h       Normal    Killing                   pod/kubia-bcf9bb974-vljhk    Stopping container nodejs
2d21h       Normal    SuccessfulCreate          replicaset/kubia-bcf9bb974   Created pod: kubia-bcf9bb974-5z9g8
2d21h       Normal    SuccessfulCreate          replicaset/kubia-bcf9bb974   Created pod: kubia-bcf9bb974-89b8z
2d21h       Normal    SuccessfulCreate          replicaset/kubia-bcf9bb974   Created pod: kubia-bcf9bb974-l9dkq
2d21h       Normal    SuccessfulCreate          statefulset/kubia            create Pod kubia-0 in StatefulSet kubia successful
2d21h       Normal    ScalingReplicaSet         deployment/kubia             Scaled up replica set kubia-bcf9bb974 to 3
2d21h       Warning   FailedToUpdateEndpoint    endpoints/kubia              Failed to update endpoint default/kubia: Operation cannot be fulfilled on endpoints "kubia": the object has been modified; please apply your changes to the latest version and try again
24m         Normal    Starting                  node/minikube                Starting kubelet.
24m         Normal    NodeHasSufficientMemory   node/minikube                Node minikube status is now: NodeHasSufficientMemory
24m         Normal    NodeHasNoDiskPressure     node/minikube                Node minikube status is now: NodeHasNoDiskPressure
24m         Normal    NodeHasSufficientPID      node/minikube                Node minikube status is now: NodeHasSufficientPID
24m         Normal    NodeAllocatableEnforced   node/minikube                Updated Node Allocatable limit across pods
24m         Normal    RegisteredNode            node/minikube                Node minikube event: Registered Node minikube in Controller
2d20h       Normal    Scheduled                 pod/srvlookup                Successfully assigned default/srvlookup to minikube
2d20h       Normal    Pulling                   pod/srvlookup                Pulling image "tutum/dnsutils"
2d20h       Normal    Pulled                    pod/srvlookup                Successfully pulled image "tutum/dnsutils" in 44.183404157s
2d20h       Normal    Created                   pod/srvlookup                Created container srvlookup
2d20h       Normal    Started                   pod/srvlookup                Started container srvlookup
```
## 11.6 运行高可用集群
**控制平面组件使用的了领导选举机制**
领导选举的时候不需要这些组件相互通信。他们用了一个乐观锁类似的概念。其中包含了一个holderIdentity的字段，这个字段包含了自己的名字。第一个成功将自己名字写入该字段的实例成为领导者。一旦成为领导者，必须更新资源。其他实例就知道他是否存活。如果宕机了，其他实例就会发现资源有一段时间没有更新了。就会尝试把自己的名字写到资源中，尝试成为领导者。