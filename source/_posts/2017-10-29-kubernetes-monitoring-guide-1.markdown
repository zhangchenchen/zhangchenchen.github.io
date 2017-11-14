---
layout: post
title:  "kubernetes-- kubernetes 监控指南（一）"
date:   2017-10-29 18:03:10
tags: 
  - kubernetes
---


关于k8s的监控，官方文档有一个系列[Kubernetes: a monitoring guide](http://blog.kubernetes.io/2017/05/kubernetes-monitoring-guide.html),涵盖了大部分内容，如果觉得费时可以直接参阅[这篇文章](https://blog.ruhm.me/post/kubernetes-monitoring/),本文主要是偏重Heapster & InfluxDB & Grafana 方案的实践。


## Heapster & InfluxDB & Grafana 

Heapster & InfluxDB & Grafana的组合是kubernetes监控的官方推荐组合，官方dashboard的概览中就用到了这三个工具来展示监控数据。这三个组合中，
- Heapster 作为metric数据的聚合和处理
- InfluxDB是一个时间序列数据库，用来存储Heapster传过来的metric数据
- Grafana是一个dashboard的可视化展现工具。


### Heapster 介绍

Heapster会收集集群中的node，namespace，pod等级别的metric信息，对这些数据聚合之后存储到指定的后端存储系统中。Heapster是通过访问node上的kubelet的API来获取metric数据，而kubelet中聚合了cAdvisor这个工具采集当前节点的所有容器的性能数据。目前Heapster支持的后端数据库包括memory、InfluxDB、BigQuery、 Google Cloud Monitoring 和 Google Cloud Logging等。
Heapster收集到的metric数据可以通过restAPI访问，主要是CPU和内存数据，包括集群级别，node级别，namespace级别，pod级别，容器级别的metric数据。

### Grafanna 介绍

Grafanna是一个比较知名的dashboard可视化工具,可以将时序数据通过检索展现成图标或曲线等形式，通过插件机制支持多种后端数据源，比如InfluxDB, Graphite, Elasticsearch, Prometheus等。


### InfluxDB 介绍

InfluxDB是基于LevelDB ，为了优化写请求比较多的情况而实现的一种时间序列数据库，每一条数据都带有时间戳属性，主要用于实时数据采集，时间跟踪记录等。

如下图，便是kubernetes监控的总体架构图：

![k8s-monitor-arc](http://oeptotikb.bkt.clouddn.com/2017-10-31-monitoring-architecture.png)



## 安装部署

- 首先要有一个k8s集群，然后clone heapster的yaml文件。

```bash
git clone https://github.com/kubernetes/heapster.git
```

- 根据yaml文件创建对应的服务，yaml文件有3个，分别对应Heapster & InfluxDB & Grafana，查看yaml文件，其实就是创建了一系列serviceAccount，DeployMent，Service。注：如果是生产环境，注意修改InfluxDB的存储由暂时性volume改为持久volume（将emptyDir: {}换成挂载磁盘或其他存储方式）。

```bash
cd ./heapster/deploy/kube-config/influxdb 
kubectl create -f  ./*
```
- 创建完成后，查看对应的pod是否起来了，如果没有起来在排错。对于kubeadm部署的集群，默认是采用tls双向认证，且采用RBAC的授权机制，所以还需要创建rolebinding.

```bash
kubectl create -f  ./heapster/deploy/kube-config/rbac/heapster-rbac.yaml
```

## 配置参数解析

看下heapster的yaml文件：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: gcr.io/google_containers/heapster-amd64:v1.4.0
        imagePullPolicy: IfNotPresent
        command:
        - /heapster
        - --source=kubernetes:https://kubernetes.default
        - --sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
---
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
```

可以看到创建了一个serviceAccount，用于与api-server通信，创建了一个Deployment，拉取相关镜像，可以看到执行heapster命令时，有两个参数，一个source指定metric的来源，https://kubernetes.default就是默认的k8s集群访问地址，可以通过kubectl get service获得对应IP。sink指定要存储的数据地址，这里就是influxDB的地址。最后创建对应的service。
其他服务类似。
 
## 服务调用与实践

这三个服务都有对应的restAPI可以调用，执行kubectl cluster-info 可以获取对应的地址。

![cluster-info](http://oeptotikb.bkt.clouddn.com/2017-10-31-cluster-info.jpg)

可以看出都是走的api-server的端口，应该是api-server调的这三个服务。

如果直接在浏览器里输入路径，会出现认证不通过错误：“User "system:anonymous" cannot proxy services in the namespace "kube-system".”。这时因为rbac的权限认证导致，可以在浏览器中导入admin证书，以admin的角色进行访问。也可以通过kubectl proxy，在本地访问，也可以设置api-server的insecure-port，通过不安全端口进行访问，因为不安全端口不会进行认证和授权。

### 通过 insecure-port 访问 heapster

- 修改api-server 的yaml文件，添加insecure-port,insecure-bind-address

```yaml
metadata:
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    ..................
    - --insecure-port=6444 # 添加不安全端口
    - --insecure-bind-address=0.0.0.0 # 绑定不安全端口地址
    ..................
```

- 由不安全端口构造url,如下示例(关于如何构造url参考[使用Heapster获取kubernetes集群对象的metric数据](https://jimmysong.io/posts/using-heapster-to-get-object-metrics/))：

```bash
curl http://172.16.21.250:6444/api/v1/proxy/namespaces/kube-system/services/heapster/api/v1/model/namespaces/kube-system/metrics/memory/usage
{
  "metrics": [  
   {
    "timestamp": "2017-11-01T15:18:00Z",
    "value": 1124192256
   },
   {
    "timestamp": "2017-11-01T15:19:00Z",
    "value": 1120612352
   },
   {
    "timestamp": "2017-11-01T15:20:00Z",
    "value": 1123553280
   },
   {
    "timestamp": "2017-11-01T15:21:00Z",
    "value": 1121275904
   }
  ],
  "latestTimestamp": "2017-11-01T15:21:00Z"
 }
```


### 访问 Grafana 界面

Grafana也提供rest api，url 的构建同上，可以参考[HTTP API Reference](http://docs.grafana.org/http_api/)。除此之外，grafana还有一个UI可以访问，在Grafana的yaml文件中添加type：NodePort,然后就可以通过ip:NodePort访问该UI。

![grafana-ui](http://oeptotikb.bkt.clouddn.com/grafana-20171101154016.jpg)

通过该UI界面，设置查询SQL，就可以定制自己需要的图表，参考[How to Utilize the “Heapster + InfluxDB + Grafana” Stack in Kubernetes for Monitoring Pods](https://blog.kublr.com/how-to-utilize-the-heapster-influxdb-grafana-stack-in-kubernetes-for-monitoring-pods-4a553f4d36c9)


以上就是Heapster & InfluxDB & Grafana 实现k8s监控的内容，这套方案的优点就是部署简单，与k8s结合的很好，缺点也比较明显，heapster是专为k8s设计的，没有通用性，且没有alert机制（当然可以通过设置grafana，但是需要自己做镜像）。除了这一套方案外 ，还有一种广泛使用的方案：[prometheus + Grafana](https://zhangchenchen.github.io/2017/11/09/kubernetes-monitoring-guide-2/)，这套方案相对比较通用，除了k8s还可以对接其他系统（比如运行在k8s里面的数据库集群等），且有报警模块alertmanager，缺点就是配置相对比较麻烦，不过为了以后的方便，还是建议采用这种方案。


## 参考文章


[Kubernets监控 Heapster+InfluxDB+Grafana ](http://blog.codecp.org/2016/07/07/Kubernets%E7%9B%91%E6%8E%A7%20Heapster+InfluxDB+Grafana/)

[Kubernetes: a monitoring guide](http://blog.kubernetes.io/2017/05/kubernetes-monitoring-guide.html)

[Monitoring Kubernetes](https://sysdig.com/blog/monitoring-kubernetes-with-sysdig-cloud/)

[How to Utilize the “Heapster + InfluxDB + Grafana” Stack in Kubernetes for Monitoring Pods](https://blog.kublr.com/how-to-utilize-the-heapster-influxdb-grafana-stack-in-kubernetes-for-monitoring-pods-4a553f4d36c9)

[使用Heapster获取kubernetes集群对象的metric数据](https://jimmysong.io/posts/using-heapster-to-get-object-metrics/)

[UBERNETES 集群监控方案研究](https://blog.ruhm.me/post/kubernetes-monitoring/)

[使用Prometheus完成Kubernetes集群监控](https://jishu.io/kubernetes/kubernetes-monitoring-with-prometheus/)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***