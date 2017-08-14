---
layout: post
title:  "Kubernete-- 利用kubeadm 搭建一个kubernate集群"
date:   2017-08-14 15:38:10
tags: 
  - kubernete
---


## 目标

- 利用 kubeadm 搭建一个四节点的k8s测试集群
- 利用harbor搭建一个单节点的私有镜像仓库
- k8s集群与私有镜像仓库整合

## 前期准备

准备以下5个节点，一个为k8s的master节点，3个为node节点，最后一个作为私有仓库镜像，系统为centos7.2：

![five-nodes](http://7xrnwq.com1.z0.glb.clouddn.com/2017-08-14-node.png)

注：k8s的安装方式有很多，kubeadm安装方式是独立节点安装的官方推荐方式，简单可重复，但不适用于生产环境，因为没有做HA，不过可以在安装完之后继续优化做HA，参考[一步步打造基于Kubeadm的高可用Kubernetes集群](http://tonybai.com/2017/05/15/setup-a-ha-kubernetes-cluster-based-on-kubeadm-part1/),后续会跟进这一块。


## kubernete 集群安装



### k8s所有节点需要执行的操作

所有节点都要安装以下组件：

```bash
docker：容器运行时，被Kubernetes依赖
kubelet：Kubernetes核心组件，运行在集群中的所有节点上，用来启动容器和pods
kubectl：命令行工具，k8s客户端，用来控制集群，只需要安装到kube-master上,当然，也可以安装到其他节点，然后配置指定master。
kubeadm：集群安装工具
```

首先，安装docker,k8s官方建议版本为1.12，1.13以及17.03+版本还没有测试。所以这里也安装1.12版本。

```bash
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

setenforce 0

yum update -y 

yum install -y docker-engine-1.12.6 docker-engine-selinux-1.12.6

systemctl enable docker && systemctl start docker
```

注：这里有个小坑，就是k8s dashboard在某些版本RH内核下会启动失败，参考[issue](
https://github.com/rancher/rancher/issues/7436)。

接下来，安装kubectl, kubelet, kubeadm以及一些依赖包。

先把依赖包装上：
```bash
yum install -y ebtables socat
```

kubectl 的安装比较简单，参考[Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/),可以直接下载可执行文件然后添加权限，扔到master节点的/usr/local/bin/目录下即可，注意版本要与k8s版本匹配。

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

因为kubelet, kubeadm的rpm安装包在gce上，需要翻墙。

如果服务器可以翻墙，可以直接通过yum命令安装：
```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm
systemctl enable kubelet && systemctl start kubelet
```

如果不能翻墙，只能先下载下来，然后安装，需要安装的rpm包url地址可以在这个网页中找到：
```bash
curl https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64/repodata/primary.xml
```

我们这里只需要安装三个rpm包，kubeadm, kubelet以及kubernetes-cni，可以直接搜索上面的网页然后找到合适版本的rpm包。我们这里安装最新版本1.7.3,对应的地址如下：

```bash
https://packages.cloud.google.com/yum/pool/f7ec56b0f36a81c0f91bcf26e05f23088082b468b77dac576dc505444dd8cd48-kubeadm-1.7.3-1.x86_64.rpm

https://packages.cloud.google.com/yum/pool/28b76e6e1c2ec397a9b6111045316a0943da73dd5602ee8e53752cdca62409e6-kubelet-1.7.3-1.x86_64.rpm

https://packages.cloud.google.com/yum/pool/e7a4403227dd24036f3b0615663a371c4e07a95be5fee53505e647fd8ae58aa6-kubernetes-cni-0.5.1-0.x86_64.rpm

```
将这三个rpm包打包上传到四个节点上，并安装：

```bash
tar xzvf /tmp/kubernetes-el7-x86_64.tar.gz
kubernetes-el7-x86_64/
kubernetes-el7-x86_64/567600102f687e0f27bd1fd3d8211ec1cb12e71742221526bb4e14a412f4fdb5-kubernetes-cni-0.3.0.1-0.07a8a2.x86_64.rpm
kubernetes-el7-x86_64/5612db97409141d7fd839e734d9ad3864dcc16a630b2a91c312589a0a0d960d0-kubeadm-1.6.0-0.alpha.0.2074.a092d8e0f95f52.x86_64.rpm
kubernetes-el7-x86_64/8a299eb1db946b2bdf01c5d5c58ef959e7a9d9a0dd706e570028ebb14d48c42e-kubelet-1.5.1-0.x86_64.rpm
kubernetes-el7-x86_64/93af9d0fbd67365fa5bf3f85e3d36060138a62ab77e133e35f6cadc1fdc15299-kubectl-1.5.1-0.x86_64.rpm

cd kubernetes-el7-x86_64/

rpm -ivh *

systemctl enable kubelet && systemctl start kubelet
```

接下来开始基于Kubeadm 创建k8s集群，不过在开始之前，我们先准备下需要用到的镜像，因为kubeadm创建的k8s集群中的kub-api, kube-scheduler, kube-proxy, kube-controller-manager,etcd等服务都是直接拉取镜像跑在k8s集群中，为了避免安装过程中下载镜像浪费太多时间，这里先把镜像下载好。各个版本需要下载的镜像版本也不一样。参考如下：

![k8s-image](http://7xrnwq.com1.z0.glb.clouddn.com/2017-08-14-k8s-image.png)

我们直接用的最新版1.7.3，如果服务器可以翻墙，直接拉取镜像：

```bash
images=(kube-proxy-amd64:v1.7.3 kube-scheduler-amd64:v1.7.3 kube-controller-manager-amd64:v1.7.3 kube-apiserver-amd64:v1.7.3 etcd-amd64:3.0.17 k8s-dns-sidecar-amd64:1.14.4 pause-amd64:3.0 k8s-dns-kube-dns-amd64:1.14.4 k8s-dns-dnsmasq-nanny-amd64:1.14.4)
for imageName in ${images[@]} ; do
  docker pull gcr.io/google_containers/$imageName
done
```

如果不能翻墙，可以先翻墙下载下来，然后上传到dockerhub上，再下载下来。


### master 节点安装

在master节点执行：

```bash
kubeadm init
```
执行完成后，会输出一个token，node节点安装时会用到。


### node 节点安装

在各node节点执行：

```bash
 kubeadm join --token=976234.e91451d4305bc282 172.16.21.53 # node IP
```

全部执行完成后，在master节点验证：

```bash
[root@k8s-master ~# kubectl get nodes
NAME                   STATUS         AGE
k8s-master.novalocal   Ready,master   4d
k8s-node1.novalocal    Ready          4d
k8s-node2.novalocal    Ready          4d
k8s-node3.novalocal    Ready          4d
```

### 部署pod网络

这里选择Weave Net。

```bash
[root@k8s-master ~]# kubectl apply -f https://git.io/weave-kube
```

等待一段时间，利用下列命令查看部署情况。

```bash
[root@k8s-master ~]# kubectl get pods --all-namespaces
```


### 部署sock-shop微服务demo

```bash
[root@k8s-master ~]# kubectl create namespace sock-shop
[root@k8s-master ~]# kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"

```
查看服务部署情况：
```bash
[root@k8s-master ~]# kubectl get pods -n sock-shop
```
访问172.16.21.34:30001验证。


## Harbor 安装

比较简单，参考[harbor doc](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)。

注意：docker 默认连接镜像使用https，而harbor默认安装是走的http，所以需要修改/etc/docker/daemon.json，添加

```bash
{
    "registry-mirrors": ["<your accelerate address>"],
    "insecure-registries": ["172.16.21.44"]
}
```

## k8s 添加私有镜像

官方给出了三种解决方案：

- 在node节点配置私有镜像的认证登录文件，其实相当于在node本地执行docker login后，在/.docker目录下生成的一个config.json文件。

```bash
# cat ~/.docker/config.json
{
    "auths": {
        "registry.cn-hangzhou.aliyuncs.com/xxxx/rbd-rest-api": {
            "auth": "xxxxyyyyzzzz"   #一个base64编码结果，不太安全
        }
    }
}
```

这种方法比较繁琐，而且不安全，不推荐。

- 利用kubectl创建docker-registry的secret

```bash
kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

- 通过secret yaml文件创建pull image所用的secret,其实跟上述方法类似，不过是用yaml文件创建的secret.

```bash
# cat myregistrykey.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
  namespace: awesomeapps
data:
  .dockerconfigjson: {base64 -w 0 ~/.docker/config.json}
type: kubernetes.io/dockerconfigjson
```

其中，dockerconfigjson后面的数据就是docker login后生成的config.json文件的base64编码输出（-w 0让base64输出在单行上，避免折行）

```bash
kubectl create -f myregistrykey.yaml
```



## 参考文章


[k8s-doc-Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

[CentOS 7 安装Kubernetes 1.5.3 集群(本地安装)](http://yoyolive.com/2017/02/27/Kubernetes-1-5-3-Local-Install/)

[Kubernetes从Private Registry中拉取容器镜像的方法]http://tonybai.com/2016/11/16/how-to-pull-images-from-private-registry-on-kubernetes-cluster/?utm_source=rss)

[k8s-doc-Images](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***