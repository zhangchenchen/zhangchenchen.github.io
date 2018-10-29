---
layout: post
title:  "Kubernetes-- 漫谈kubernetes 中的认证 & 授权 & 准入机制"
date:   2017-08-17 15:37:30
tags: 
  - kubernetes
---


## 概览

首先需要了解这三种机制的区别：简单来说，认证(Authenticating)是对客户端的认证，通俗点就是用户名密码验证，授权(Authorization)是对资源的授权，k8s中的资源无非是容器，最终其实就是容器的计算，网络，存储资源，当一个请求经过认证后，需要访问某一个资源（比如创建一个pod），授权检查都会通过访问策略比较该请求上下文的属性，（比如用户，资源和Namespace），根据授权规则判定该资源（比如某namespace下的pod）是否是该客户可访问的。准入(Admission Control)机制是一种在改变资源的持久化之前（比如某些资源的创建或删除，修改等之前）的机制。
在k8s中，这三种机制如下图：

![k8s-authorization](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-08-17-k8s-authorition.png)

k8s的整体架构也是一个微服务的架构，所有的请求都是通过一个GateWay，也就是kube-apiserver这个组件（对外提供REST服务），由图中可以看出，k8s中客户端有两类，一种是普通用户，一种是集群内的Pod，这两种客户端的认证机制略有不同，后文会详述。但无论是哪一种，都需要依次经过认证，授权，准入这三个机制。

## kubernetes 中的认证机制

需要注意的是，kubernetes虽然提供了多种认证机制，但并没有提供user 实体信息的存储，也就是说，账户体系需要我们自己去做维护。当然，也可以接入第三方账户体系（如谷歌账户），也可以使用开源的keystone去做整合。kubernetes 支持多种认证机制，可以配置成多个认证体制共存，这样，只要有一个认证通过，这个request就认证通过了。下面介绍下官网列举的几种常见认证机制：

### X509 Client Certs

也叫作双向数字证书认证，HTTPS证书认证，是基于CA根证书签名的双向数字证书认证方式，是所有认证方式中最严格的认证。默认在kubeadm创建的集群中是enabled的，可以在master node上查看kube-apiserver的pod配置文件：


```bash
# cat /etc/kubernetes/manifests/kube-apiserver.json
.................
containers": [
      {
        "name": "kube-apiserver",
        "image": "gcr.io/google_containers/kube-apiserver-amd64:v1.5.2",
        "command": [
          "kube-apiserver",
          "--insecure-bind-address=127.0.0.1",
          "--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota",
          "--service-cluster-ip-range=10.96.0.0/12",
          "--service-account-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--client-ca-file=/etc/kubernetes/pki/ca.pem",
          "--tls-cert-file=/etc/kubernetes/pki/apiserver.pem",
          "--tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--token-auth-file=/etc/kubernetes/pki/tokens.csv",
          "--secure-port=6443",
          "--allow-privileged",
          "--advertise-address=192.168.61.100",
          "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
          "--anonymous-auth=false",
          "--etcd-servers=http://127.0.0.1:2379"
        ],
```

相关的三个启动参数：

- client-ca-file: 指定CA根证书文件为/etc/kubernetes/pki/ca.pem，内置CA公钥用于验证某证书是否是CA签发的证书
- tls-private-key-file: 指定ApiServer私钥文件为/etc/kubernetes/pki/apiserver-key.pem
- tls-cert-file：指定ApiServer证书文件为/etc/kubernetes/pki/apiserver.pem

只要有这三个启动参数，就说明开启了https的认证方式，这时，如果在集群外访问 https://masterIP:6443/api 会提示Unauthorized，只有在客户端配置相关认证才可以访问,客户端的认证证书生成与操作可以参考[Creating Certificates](https://kubernetes.io/docs/admin/authentication/#x509-client-certs)。证书的生成是kubeadm使用openssl自动生成的，如果是手动配置双向认证，相对比较麻烦，主要配置流程如下：
- 生成根证书、API Server服务端证书、服务端私钥、各个组件所用的客户端证书和客户端私钥。
- 修改 Kubernetes 各个服务进程的启动参数，启用双向认证模式.

详细配置可以参考[Kubernetes集群安全配置案例](http://www.cnblogs.com/breg/p/5923604.html)

注意，在启动参数中还有一个参数：--insecure-bind-address=127.0.0.1，这个参数主要用与master node上的其他核心组件，比如kube-scheduler，kube-controller-manager通过masterIP:8080与APIserver直接通信，而不用通过双向认证。这一点可以从他们的启动参数--master=127.0.0.1:8080看出。



### Static Token File

静态token文件认证，同样，在kubeadm创建的集群中也是默认enabled的，比如，上面的apiserver启动参数中，我们可以看到有参数 ：--token-auth-file=/etc/kubernetes/pki/tokens.csv ，这个静态token文件的格式为 token,user,uid,"group1,group2,group3"，如下示例：

```bash
# cat /etc/kubernetes/pki/tokens.csv
7db2f1c02d721320,kubeadm-node-csr,0615e0ac-7d70-11e7-ad94-fa163eb9dfdd,system:kubelet-bootstrap
```
客户端请求的时候需要在http header中加入："Authorization: Bearer THETOKEN"，如下实例：

```bash
curl -k --header "Authorization: Bearer 7db2f1c02d721320" https://192.168.21.34:6443/api
```

或者使用brctl:

```bash
kubectl --server=https://192.168.21.34:6443 \
--token=7db2f1c02d721320 \
--insecure-skip-tls-verify=true \
cluster-info
```

注意，如果该静态token文件更改的话，需要重启apiserver。


### Bootstrap Tokens

bootstrap token认证目前处于alpha阶段，目前主要是kubeadm创建k8s集群时使用。使用这种认证方式，k8s会动态的管理一种type为bootstrap token的token，这些token作为secret放在kube-system namespace中。controller-manager中的tokencleaner controller会在bootstrap token 过期时进行删除。
使用这种认证方式，apiserver的启动参数中需要有--experimental-bootstrap-token-auth，Controller Manager的启动参数中有--controllers=*,tokencleaner 类似参数。



### Static Password File

比较简单，kubeadm默认没有开启，生产环境也不建议使用。
apiserver启动参数指定--basic_auth_file=/etc/kubernetes/basic_auth。然后在指定的文件中加入用户名密码等就可以了，文件格式为password,user,uid,"group1,group2,group3"。


### Service Account Tokens

Service Account Token 是一种比较特殊的认证机制，适用于上文中提到的pod内部服务需要访问apiserver的认证情况，默认enabled。
还是看上文中apiserver 的启动配置参数有--service-account-key-file=/etc/kubernetes/pki/apiserver-key.pem，如果没有指明文件，默认使用–tls-private-key-file的值，即API Server的私钥。
service accout本身是作为一种资源在k8s集群中，我们可以通过命令行获取：

```bash
[root@k8s-master pki]# kubectl get serviceaccount --all-namespaces
NAMESPACE     NAME        SECRETS   AGE
default       default     1         7d
for-test      default     1         3d
kube-system   default     1         7d
kube-system   weave-net   1         7d
sock-shop     default     1         7d
```
可以看到k8s集群为所有的namespace创建了一个默认的service account，利用命令describe会发现service account只是关联了一个secret作为token，也就是service-account-token。

```bash
[root@k8s-master pki]# kubectl describe serviceaccount/default -n kube-system
Name:           default
Namespace:      kube-system
Labels:         <none>

Image pull secrets:     <none>

Mountable secrets:      default-token-nbldr

Tokens:                 default-token-nbldr

[root@k8s-master pki]#  kubectl get secret default-token-nbldr -o yaml -n kube-system
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFM0
  ........................略....................
  namespace: a3ViZS1zeXN0ZW0=
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJblI1Y0..................................
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 67aae699-7d70-11e7-a8a9-fa163eb9dfdd
  creationTimestamp: 2017-08-10T02:05:38Z
  name: default-token-nbldr
  namespace: kube-system
  resourceVersion: "88"
  selfLink: /api/v1/namespaces/kube-system/secrets/default-token-nbldr
  uid: 67b20b73-7d70-11e7-a8a9-fa163eb9dfdd
type: kubernetes.io/service-account-token

```

可以看到service-account-token的secret资源包含的数据有三部分：

- ca.crt，这是API Server的CA公钥证书，用于Pod中的Process对API Server的服务端数字证书进行校验时使用的；

- namespace，这是Secret所在namespace的值的base64编码：# echo -n “kube-system”|base64 => “a3ViZS1zeXN0ZW0=”

- token：该token就是由service-account-key-file的值签署(sign)生成。

这种认证方式主要由k8s集群自己管理，用户用到的情况比较少。我们创建一个pod时，默认就会将该namespace对应的默认service account token mount到Pod中，所以无需我们操作便可以直接与apiserver通信，相关示例参考[在Kubernetes Pod中使用Service Account访问API Server](http://tonybai.com/2017/03/03/access-api-server-from-a-pod-through-serviceaccount/)，当然也可以指定多个service account token,参考[Configure Service Accounts for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)。


### OpenID Connect Tokens

类似 OAuth2的认证方式，大致认证过程如下：

![openID](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/2017-08-17-k8s-openid-token.jpg)

除了以上几种认证方式外，还有几种比如Webhook Token Authentication，Keystone Password等，详情见[官网](https://kubernetes.io/docs/admin/authentication/#x509-client-certs)。


## kubernetes 中的授权机制

k8s中的授权策略也支持开启多个授权插件，只要一个验证通过即可。k8s授权处理主要是根据以下请求属性：

- user, group, extra
- API、请求方法（如get、post、update、patch和delete）和请求路径（如/api）
- 请求资源和子资源
- Namespace
- API Group

目前k8s支持的授权模式主要有以下几种：


- Node Authorization
- ABAC Authorization
- RBAC Authorization
- Webhook Authorization

### Node Authorization

1.7+版本才release的一种授权机制，通过配合NodeRestriction control准入控制插件来限制kubelet访问node，endpoint、pod、service以及secret、configmap、PV和PVC等相关的资源。配置方式为：
--authorization-mode=Node,RBAC --admission-control=...,NodeRestriction,...


### ABAC Authorization

ABAC(Attribute-based access control),使用这种模式需要配置参数：
--authorization-mode=ABAC  --authorization-policy-file=SOME_FILENAME。
这种模式的实现相对比较生硬，就是在master node保存一份policy文件，指定不用用户（或用户组）对不同资源的访问权限,当修改该文件后，需要重启apiserver,跟openstack 的ABAC类似。policy文件的格式如下：

```bash
# Alice can do anything to all resources:
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "alice",
        "namespace": "*",
        "resource": "*",
        "apiGroup": "*"
    }
}
# Kubelet can read any pods:
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "kubelet",
        "namespace": "*",
        "resource": "pods",
        "readonly": true
    }
}

# Kubelet can read and write events:
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "kubelet",
        "namespace": "*",
        "resource": "events"
    }
}
```

使用这种模式需要配置参数：
--authorization-mode=ABAC  --authorization-policy-file=SOME_FILENAME

### RBAC Authorization

RBAC（Role-Based Access Control）依然处于Beta阶段，通过启动参数--authorization-mode=RBAC，使用kubeadm安装k8s默认会enabled。
RBAC API定义了四个资源对象用于描述RBAC中用户和资源之间的连接权限：
- Role
- ClusterRole
- RoleBinding
- ClusterRoleBinding

Role是定义在某个Namespace下的资源，在这个具体的Namespace下使用。 ClusterRole与Role相似，只是ClusterRole是整个集群范围内使用的。
RoleBinding把Role绑定到账户主体Subject，让Subject继承Role所在namespace下的权限。 ClusterRoleBinding把ClusterRole绑定到Subject，让Subject集成ClusterRole在整个集群中的权限。

我们可以通过kubectl命令获取对应的Role相关资源进行增删改查：
```bash
kubectl get roles --all-namespaces

kubectl get ClusterRoles

kubectl get rolebinding --all-namespaces

kubectl get clusterrolebinding
```

API Server已经创建一系列ClusterRole和ClusterRoleBinding。这些资源对象中名称以system:开头的，表示这个资源对象属于Kubernetes系统基础设施。 也就说RBAC默认的集群角色已经完成足够的覆盖，让集群可以完全在 RBAC的管理下运行。 修改这些资源对象可能会引起未知的后果，例如对于system:node这个ClusterRole定义了kubelet进程的权限，如果这个角色被修改，可能导致kubelet无法工作。

### Webhook Authorization

用户在外部提供 HTTPS 授权服务，然后配置 apiserver 调用该服务去进行授权。apiserver配置参数：
--authorization-webhook-config-file=SOME_FILENAME
配置文件的格式跟kubeconfig的格式类似，具体参考[官方文档](https://kubernetes.io/docs/admin/authorization/webhook/)

## kubernetes 中的准入机制

Kubernetes的Admission Control实际上是一个准入控制器(Admission Controller)插件列表，发送到APIServer的请求都需要经过这个列表中的每个准入控制器插件的检查，如果某一个控制器插件准入失败，就准入失败。
控制器插件如下：

- AlwaysAdmit：允许所有请求通过
- AlwaysPullImages：在启动容器之前总是去下载镜像，相当于每当容器启动前做一次用于是否有权使用该容器镜像的检查
- AlwaysDeny：禁止所有请求通过，用于测试
- DenyEscalatingExec：拒绝exec和attach命令到有升级特权的Pod的终端用户访问。如果集中包含升级特权的容器，而要限制终端用户在这些容器中执行命令的能力，推荐使用此插件
- ImagePolicyWebhook
- ServiceAccount：这个插件实现了serviceAccounts等等自动化，如果使用ServiceAccount对象，强烈推荐使用这个插件
- SecurityContextDeny：将Pod定义中定义了的SecurityContext选项全部失效。SecurityContext包含在容器中定义了操作系统级别的安全选型如fsGroup，selinux等选项
- ResourceQuota：用于namespace上的配额管理，它会观察进入的请求，确保在namespace上的配额不超标。推荐将这个插件放到准入控制器列表的最后一个。ResourceQuota准入控制器既可以限制某个namespace中创建资源的数量，又可以限制某个namespace中被Pod请求的资源总量。ResourceQuota准入控制器和ResourceQuota资源对象一起可以实现资源配额管理。
- LimitRanger：用于Pod和容器上的配额管理，它会观察进入的请求，确保Pod和容器上的配额不会超标。准入控制器LimitRanger和资源对象LimitRange一起实现资源限制管理
- NamespaceLifecycle：当一个请求是在一个不存在的namespace下创建资源对象时，该请求会被拒绝。当删除一个namespace时，将会删除该namespace下的所有资源对象
- DefaultStorageClass
- DefaultTolerationSeconds
- PodSecurityPolicy

当Kubernetes版本>=1.6.0，官方建议使用这些插件：
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds
当Kubernetes版本>=1.4.0，官方建议使用这些插件：
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
以上是标准的准入插件，如果是自己定制的话，k8s1.7版 出了两个alpha features, Initializers 和 External Admission Webhooks，详情可以参考[Dynamic Admission Control](https://kubernetes.io/docs/admin/extensible-admission-controllers/).






## 参考文章

[Authenticating](https://kubernetes.io/docs/admin/authentication/#x509-client-certs)

[authorization](https://kubernetes.io/docs/admin/authorization/)

[Kubernetes集群安全：Api Server认证](http://blog.frognew.com/2017/01/kubernetes-api-server-authc.html)

[Kubernetes集群安全：准入控制Admission Control](http://blog.frognew.com/2017/05/kubernetes-apiserver-admission-control.html)

[Kubernetes 1.6新特性学习：RBAC授权](http://blog.frognew.com/2017/04/kubernetes-1.6-rbac.html)

[在Kubernetes Pod中使用Service Account访问API Server](http://tonybai.com/2017/03/03/access-api-server-from-a-pod-through-serviceaccount/)

[ kubernetes安全控制认证与授权(二)](http://blog.csdn.net/yan234280533/article/details/76359199)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***