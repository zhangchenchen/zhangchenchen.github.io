---
layout: post
title:  "kubernetes-- kubernetes 使用ceph rbd作为持久存储"
date:   2017-11-17 18:03:10
tags: 
  - kubernetes
  - ceph
---

## kubernetes 中的存储方案

对于有状态服务，存储是一个至关重要的问题。k8s提供了非常丰富的组件来支持存储，这里大致列一下：

- volume: 就是直接挂载在pod上的组件，k8s中所有的其他存储组件都是通过volume来跟pod直接联系的。volume有个type属性，type决定了挂载的存储是什么，常见的比如：emptyDir，hostPath，nfs，rbd，以及下文要说的persistentVolumeClaim等。跟docker里面的volume概念不同的是，docker里的volume的生命周期是跟docker紧紧绑在一起的。这里根据type的不同，生命周期也不同，比如emptyDir类型的就是跟docker一样，pod挂掉，对应的volume也就消失了，而其他类型的都是永久存储。详细介绍可以参考[Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- Persistent Volumes：顾名思义，这个组件就是用来支持永久存储的，Persistent Volumes组件会抽象后端存储的提供者（也就是上文中volume中的type）和消费者（即具体哪个pod使用）。该组件提供了PersistentVolume和PersistentVolumeClaim两个概念来抽象上述两者。一个PersistentVolume（简称PV）就是后端存储提供的一块存储空间，具体到ceph rbd中就是一个image，一个PersistentVolumeClaim（简称PVC）可以看做是用户对PV的请求，PVC会跟某个PV绑定，然后某个具体pod会在volume 中挂载PVC,就挂载了对应的PV。关于更多详细信息比如PV,PVC的生命周期，dockerfile 格式等信息参考[Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- Dynamic Volume Provisioning: 动态volume发现，比如上面的Persistent Volumes,我们必须先要创建一个存储块，比如一个ceph中的image，然后将该image绑定PV，才能使用。这种静态的绑定模式太僵硬，每次申请存储都要向存储提供者索要一份存储快。Dynamic Volume Provisioning就是解决这个问题的。它引入了StorageClass这个概念，StorageClass抽象了存储提供者，只需在PVC中指定StorageClass，然后说明要多大的存储就可以了，存储提供者会根据需求动态创建所需存储快。甚至于，我们可以指定一个默认StorageClass，这样，只需创建PVC就可以了。

## kubernetes 与ceph 整合

首先要有一个k8s集群，一个ceph集群，k8s集群的搭建与ceph集群的搭建不再赘述。

### 预备工作

- 在每个k8s node中安装ceph-common
```bash
yum install -y ceph-common
```
- 将ceph配置文件ceph.conf,ceph admin的认证文件ceph.client.admin.keyring复制到k8s node的/etc/ceph/ 目录下。

### 配置 ceph secret

```bash
grep key /etc/ceph/ceph.client.admin.keyring |awk '{printf "%s", $NF}'|base64
```
获取base64加密的key：QVFCZmdTcFRBQUFBQUJBQWNXTmtsMEFtK1ZkTXVYU21nQ0FmMFE9PQ==
利用该key创建ceph-secret,这里我们单独创建了一个test-ceph namespace,所有操作都在该namespace下。
ceph-secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: test-ceph
type: "kubernetes.io/rbd"  
data:
  key: QVFCZmdTcFRBQUFBQUJBQWNXTmtsMEFtK1ZkTXVYU21nQ0FmMFE9PQ==
```

### Persistent Volumes 测试

首先在ceph 中创建一个2G 的image，这里为了方便直接在rbd pool中创建。

```bash
rbd create test-image -s 2G --image-feature  layering
```

查看新建image信息：
```bash
[root@seed galera-cluster]# rbd info test-image
rbd image 'test-image':
        size 2048 MB in 512 objects
        order 22 (4096 kB objects)
        block_name_prefix: rbd_data.5ed8238e1f29
        format: 2
        features: layering
        flags: 
```

注：这里有个ceph的坑，在jewel版本下默认format是2，开启了rbd的一些属性，而这些属性有的内核版本是不支持的，会导致map不到device的情况，可以在创建时指定feature（我们就是这样做的）,也可以在ceph配置文件中关闭这些新属性：rbd_default_features = 2。参考[rbd无法map(rbd feature disable)](http://www.zphj1987.com/2016/06/07/rbd%E6%97%A0%E6%B3%95map-rbd-feature-disable/)。

创建PV,需要指定ceph mon节点地址，以及对应的pool，image等：

test.pv.yml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
  namespace: test-ceph
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce 
  rbd:
    monitors:
      - 172.16.21.250:6789
      - 172.16.21.251:6789
      - 172.16.21.252:6789
    pool: rbd
    image: test-image
    user: admin
    secretRef:
      name: ceph-secret
  persistentVolumeReclaimPolicy: Recycle
```

创建PVC：

 test.pvc.yml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
  namespace: test-ceph
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

查看PV,PVC，如果状态是bound那么两者绑定成功。

创建一个pod验证：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-dm
  namespace: test-ceph
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ceph-rbd-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: ceph-rbd-volume
        persistentVolumeClaim:
          claimName: test-pvc
```

进入该容器，利用dh -h 命令验证是否挂载成功。


### Dynamic Volume Provisioning测试

创建一个storageclass：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: ceph-storage
   namespace: test-ceph
provisioner: ceph.com/rbd
parameters:
  monitors: 172.16.21.250:6789,172.16.21.251:6789,172.16.21.252:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: test-ceph
  pool: rbd
  userId: admin
  userSecretName: ceph-secret
```

创建一个PVC，指定storageclass:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
 name: ceph-claim-dynamic
 namespace: test-ceph
spec:
 accessModes:
    - ReadWriteOnce
 resources:
   requests:
     storage: 20Gi
 storageClassName: ceph-storage
```

注：这里又趟到了一个大坑，如果这样就直接创建pod挂载的话会报错如下：
```bash
Error creating rbd image: executable file not found in $PATH
```

这是因为我们的k8s集群是使用kubeadm创建的，k8s的几个服务也是跑在集群静态pod中，而kube-controller-manager组件会调用rbd的api，但是因为它的pod中没有安装rbd，所以会报错，如果是直接安装在物理机中，因为我们已经安装了ceph-common，所以不会出现这个问题。我们在该[issue](https://github.com/kubernetes/kubernetes/issues/38923)下找到了解决方案。如下：

创建一个 rbd-provisioner ：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: rbd-provisioner
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:v0.1.0"
      serviceAccountName: persistent-volume-binder 
```

这样就可以直接用PVC了，创建 pod测试下：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test-dynamic
  namespace: test-ceph
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: ceph-rbd-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: ceph-rbd-dynamic-volume
        persistentVolumeClaim:
          claimName: ceph-claim-dynamic
```

### 实战：创建一个mysql-galera集群

在进入具体实战之前，先介绍一下k8s针对有状态服务推出的一个组件，statefulset(1.5之前叫做petset),statefulset与deployment,replicasets是一个级别的。不过Deployments和ReplicaSets是为无状态服务而设计。statefulset则是为了解决有状态服务的问题。它的应用场景如下：
- 稳定的持久化存储，即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
- 稳定的网络标志，即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有Cluster IP的Service）来实现。
- 有序部署，有序扩展，即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的Pod必须都是Running和Ready状态），基于init containers来实现。
- 有序收缩，有序删除（即从N-1到0）。

由应用场景可知，statefuleset特别适合mqsql,redis等数据库集群。相应的，一个statefuleset有以下三个部分：
- 用于定义网络标志（DNS domain）的Headless Service
- 用于创建PersistentVolumes的volumeClaimTemplates
- 定义具体应用的StatefulSet

以下是从github上找到的一个示例dockerfile:


首先创建headless service：

galera-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
  name: galera
  namespace: galera
  labels:
    app: mysql
spec:
  ports:
  - port: 3306
    name: mysql
  # *.galear.default.svc.cluster.local
  clusterIP: None
  selector:
    app: mysql

```

创建statefulset，这里存储直接用的storageclass指定ceph rbd,镜像下载需要科学上网。

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mysql
  namespace: galera
spec:
  serviceName: "galera"
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: install
        image: gcr.io/google_containers/galera-install:0.1
        imagePullPolicy: Always
        args:
        - "--work-dir=/work-dir"
        volumeMounts:
        - name: workdir
          mountPath: "/work-dir"
        - name: config
          mountPath: "/etc/mysql"
      - name: bootstrap
        image: debian:jessie
        command:
        - "/work-dir/peer-finder"
        args:
        - -on-start="/work-dir/on-start.sh"
        - "-service=galera"
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        volumeMounts:
        - name: workdir
          mountPath: "/work-dir"
        - name: config
          mountPath: "/etc/mysql"
      containers:
      - name: mysql
        image: gcr.io/google_containers/mysql-galera:e2e
        ports:
        - containerPort: 3306
          name: mysql
        - containerPort: 4444
          name: sst
        - containerPort: 4567
          name: replication
        - containerPort: 4568
          name: ist
        args:
        - --defaults-file=/etc/mysql/my-galera.cnf
        - --user=root
        readinessProbe:
          # TODO: If docker exec is buggy just use gcr.io/google_containers/mysql-healthz:1.0
          exec:
            command:
            - sh
            - -c
            - "mysql -u root -e 'show databases;'"
          initialDelaySeconds: 15
          timeoutSeconds: 5
          successThreshold: 2
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/
        - name: config
          mountPath: /etc/mysql
      volumes:
      - name: config
        emptyDir: {}
      - name: workdir
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: datadir
      annotations:
        volume.beta.kubernetes.io/storage-class: "ceph-web"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

以上涉及到三个镜像，前两个镜像在initContainers 下，表示这两个镜像创建的是Init Container，顾名思义，就是完成一些初始化的工作。这些 Init Container 按照定义的顺序依次执行，只有所有的Init Container 执行完后，主容器才启动。由于一个Pod里的存储卷是共享的，所以 Init Container 里产生的数据可以被主容器使用到。
这里两个initcontainer主要完成以下两个工作：安装mysql-galera等组件,生成配置文件等。


最终结果如下：

```bash
[root@seed galera-cluster]# kubectl get pods  -n galera
NAME                             READY     STATUS    RESTARTS   AGE
mysql-0                          1/1       Running   0          47m
mysql-1                          1/1       Running   0          24m
mysql-2                          1/1       Running   0          2m
```


## 参考文章

[kubernetes Ceph RBD](https://jicki.me/2017/05/09/kubernetes-ceph-rbd/#ceph-mon-)

[Dynamic Provisioning and Storage Classes in Kubernetes](http://blog.kubernetes.io/2017/03/dynamic-provisioning-and-storage-classes-kubernetes.html)

[使用Ceph RBD为Kubernetes集群提供存储卷](http://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)

[使用Ceph做持久化存储创建MySQL集群](https://jimmysong.io/kubernetes-handbook/practice/using-ceph-for-persistent-storage.html)

[Error creating rbd image: executable file not found in $PATH ](https://github.com/kubernetes/kubernetes/issues/38923)

[MySQL on Docker: Running Galera Cluster on Kubernetes](https://severalnines.com/blog/mysql-docker-running-galera-cluster-kubernetes)

[StatefulSet](https://jimmysong.io/kubernetes-handbook/concepts/statefulset.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***