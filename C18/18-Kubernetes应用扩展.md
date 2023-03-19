# 18. Kubernetes应用扩展
# 18.1 定义自定义API对象
### 18.1.1 CustomResourceDefinitions介绍
开发者只需要向k8s api服务器提交CRD对象，即可定义新的资源类型。
定义了一个类型为Website的资源。
imaginary-kubia-website.yaml
```
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```
用创建命令，会发现这个资源是无法被识别的。
```
k create -f imaginary-kubia-website.yaml
```
因此在创建自定义资源之前，需要确保k8s能够识别他们
**创建一个CRD**对象
想要使用k8s接受你的自定义网站资源实例，需要用一下代码清单所表示的格式向API服务提交CRD
website-crd.yaml
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com
spec:
  scope: Namespaced
  group: extensions.example.com
  version: v1
  names:
    kind: Website
    singular: website
    plural: websites
```
```
k create -f website-crd-definition.yaml
```
**创建自定义资源实例**
kubia-website.yaml
```
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```
```
k create f kubia-website.yaml
```
## 18.3 基于Kubernetes搭建平台
K8s被大量的新一代Paas产品采用。基于K8s构建的著名的Paas平台包括Deis Worflow和Red Hat的OpenShit
### 18.3.1 Red Hat OpenShitf平台
Red Hat OpenShitf注重开发者的体验。他最想要实现的目标就是帮助开发者实现程序快速开发，便捷部署，轻松扩展和长期维护。
**试用openshitf**
可以尝试Minishitf他是Minikube等效的,还可以在http://manage.openshit.com上尝试免费的多租户托管解决方案-openShitf Online Starter.
### 18.3.2 Deis Workflow与Helm
这是微软收购的Deis。
再见