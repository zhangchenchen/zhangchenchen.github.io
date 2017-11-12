---
layout: post
title:  "kubernetes-- kubernetes 监控指南（二）"
date:   2017-11-09 10:03:10
tags: 
  - kubernetes
---


之前一篇博客介绍了k8s的官方监控方案，相对比较简单，因为缺乏通用性，所以一般生产环境还是用另一种方案，也就是本篇博客要介绍的，Prometheus & Grafana.


## Prometheus & Grafana

### Prometheus介绍

Prometheus 是一个开源的系统监控预警工具，由Soundcloud开发并开源，目前是一个独立的项目，已经有很多公司采用，主要特点有以下几个：

- 多维度数据模型
- 灵活的类SQL查询语句
- 不依赖分布式存储，单个自主的服务器节点。
- 通过基于HTTP的pull方式采集时序数据。
- 对于push型的数据可以通过中间网关来暂时存储支持
- 通过服务发现或者静态配置来发现数据搜集对象。
- 支持多种多样的图表和界面展示，比如Grafana等。

不过Prometheus也不是万能的，他不适于一些实时要求比较高的监控项目，且因为是单机存储，存储量有限，可以的监控量局限存储时间。prometheus集成了leveldb，一个能高效插入数据的数据库，在ssd盘下io占用比较高，可能会有大量数据堆积内存，内存占用率大（可以进行适当配置）。

Prometheus的整体架构如下：

![prometheus-arch](http://7xrnwq.com1.z0.glb.clouddn.com/201711-12-prometheous-arch.jpg)

由上图可以看出Prometheus的几个重要组件：
- Prometheus Server：主要负责数据采集，存储，提供PromQL查询语言的支持。
- Push Gateway: 支持临时性Job主动推送数据的中间网关。
- Jobs/Exporter: 数据采集组件,从目标处搜集数据，并将其转化为Prometheus支持的格式,然后等待Prometheus Server对数据的拉取。
- Alertmanager: 警告管理器，用来进行报警。

Prometheus的工作原理大致是这样的：Prometheus Server定时去拉取暴露出metric数据的http接口（一般由exporter实现），该接口可通过配置文件、Zookeeper、Consul等方式配置。拉取到数据后，会进行一定规则的数据清洗，然后存储到时间序列数据库中，可以通过PromQL和其他API，比如grafana可视化地展示收集的数据。对于主动推送型的数据，可以先放到PushGateway中，然后等待Server拉取。


## Prometheus安装部署

有一些Prometheus的安装项目已经将所有的yaml文件写好了，比如[prometheus-operator](https://github.com/coreos/prometheus-operator),为了便于理解，本文还是一步步部署。

### 创建 一个独立的namesapce

monitoring-namespace.yaml
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

### 创建RBAC相关配置

rbac.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources:
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  verbs: ["list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - daemonsets
  - deployments
  - replicasets
  verbs: ["list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-k8s
  namespace: monitoring
```


### 创建node-exporter用于收集节点信息

exporter-daemonset.yaml
```bash
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: prometheus-node-exporter
  namespace: monitoring
  labels:
    app: prometheus
    component: node-exporter
spec:
  template:
    metadata:
      name: prometheus-node-exporter
      labels:
        app: prometheus
        component: node-exporter
    spec:
      containers:
      - image: prom/node-exporter:v0.14.0
        name: prometheus-node-exporter
        ports:
        - name: prom-node-exp
          #^ must be an IANA_SVC_NAME (at most 15 characters, ..)
          containerPort: 9100
          hostPort: 9100
      hostNetwork: true
      hostPID: true
```
创建对应的service：
exporter-service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: prometheus-node-exporter
  namespace: monitoring
  labels:
    app: prometheus
    component: node-exporter
spec:
  clusterIP: None
  ports:
    - name: prometheus-node-exporter
      port: 9100
      protocol: TCP
  selector:
    app: prometheus
    component: node-exporter
  type: ClusterIP
```
 通过以下命令查看是否创建成功：
 ```bash
kubectl -n monitoring get daemonset,svc
 ```

### 创建kube-state-metrics

用于收集k8s的metric信息。

kube-state-metrics-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: gcr.io/google_containers/kube-state-metrics:v0.5.0
        ports:
        - containerPort: 8080
```
创建对应的service:
kube-state-metrics-deployment-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  ports:
  - name: kube-state-metrics
    port: 8080
    protocol: TCP
  selector:
    app: kube-state-metrics
```
查看对应pod创建成功后，继续下一步，如果出现错误，多半是image pull不成功，可以科学上网pull下来。

### 创建node-directory-size-metrics

主要用于读取节点目录，获取磁盘使用metric数据。
node-directory-size-metrics-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-directory-size-metrics
  namespace: monitoring
  annotations:
    description: |
      This `DaemonSet` provides metrics in Prometheus format about disk usage on the nodes.
      The container `read-du` reads in sizes of all directories below /mnt and writes that to `/tmp/metrics`. It only reports directories larger then `100M` for now.
      The other container `caddy` just hands out the contents of that file on request via `http` on `/metrics` at port `9102` which are the defaults for Prometheus.
      These are scheduled on every node in the Kubernetes cluster.
      To choose directories from the node to check, just mount them on the `read-du` container below `/mnt`.
spec:
  template:
    metadata:
      labels:
        app: node-directory-size-metrics
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9102'
        description: |
          This `Pod` provides metrics in Prometheus format about disk usage on the node.
          The container `read-du` reads in sizes of all directories below /mnt and writes that to `/tmp/metrics`. It only reports directories larger then `100M` for now.
          The other container `caddy` just hands out the contents of that file on request on `/metrics` at port `9102` which are the defaults for Prometheus.
          This `Pod` is scheduled on every node in the Kubernetes cluster.
          To choose directories from the node to check just mount them on `read-du` below `/mnt`.
    spec:
      containers:
      - name: read-du
        image: giantswarm/tiny-tools
        imagePullPolicy: Always
        # FIXME threshold via env var
        # The
        command:
        - fish
        - --command
        - |
          touch /tmp/metrics-temp
          while true
            for directory in (du --bytes --separate-dirs --threshold=100M /mnt)
              echo $directory | read size path
              echo "node_directory_size_bytes{path=\"$path\"} $size" \
                >> /tmp/metrics-temp
            end
            mv /tmp/metrics-temp /tmp/metrics
            sleep 300
          end
        volumeMounts:
        - name: host-fs-var
          mountPath: /mnt/var
          readOnly: true
        - name: metrics
          mountPath: /tmp
      - name: caddy
        image: dockermuenster/caddy:0.9.3
        command:
        - "caddy"
        - "-port=9102"
        - "-root=/var/www"
        ports:
        - containerPort: 9102
        volumeMounts:
        - name: metrics
          mountPath: /var/www
      volumes:
      - name: host-fs-var
        hostPath:
          path: /var
      - name: metrics
        emptyDir:
          medium: Memory
```

### 创建prometheus 配置文件
创建prometheus的主要配置文件。
prometheus-config-map.yaml
```yaml
apiVersion: v1
data:
  prometheus.yaml: |
    global:
      scrape_interval: 10s
      scrape_timeout: 10s
      evaluation_interval: 10s
    rule_files:
      - "/etc/prometheus-rules/*.rules"
    scrape_configs:

      # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L37
      - job_name: 'kubernetes-nodes'
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: '(.*):10250'
            replacement: '${1}:10255'
            target_label: __address__

      # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L79
      - job_name: 'kubernetes-endpoints'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
            action: replace
            target_label: __scheme__
            regex: (https?)
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+)(?::\d+);(\d+)
            replacement: $1:$2
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            action: replace
            target_label: kubernetes_name

      # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L119
      - job_name: 'kubernetes-services'
        metrics_path: /probe
        params:
          module: [http_2xx]
        kubernetes_sd_configs:
          - role: service
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
            action: keep
            regex: true
          - source_labels: [__address__]
            target_label: __param_target
          - target_label: __address__
            replacement: blackbox
          - source_labels: [__param_target]
            target_label: instance
          - action: labelmap
            regex: __meta_kubernetes_service_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_service_name]
            target_label: kubernetes_name

      # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L156
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: (.+):(?:\d+);(\d+)
            replacement: ${1}:${2}
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name
          - source_labels: [__meta_kubernetes_pod_container_port_number]
            action: keep
            regex: 9\d{3}
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: prometheus-core
  namespace: monitoring
```

### 创建 prometheus-rules

prometheus-rules.yaml 
```yaml
apiVersion: v1
data:
  cpu-usage.rules: |
    ALERT NodeCPUUsage
      IF (100 - (avg by (instance) (irate(node_cpu{name="node-exporter",mode="idle"}[5m])) * 100)) > 75
      FOR 2m
      LABELS {
        severity="page"
      }
      ANNOTATIONS {
        SUMMARY = "{{$labels.instance}}: High CPU usage detected",
        DESCRIPTION = "{{$labels.instance}}: CPU usage is above 75% (current value is: {{ $value }})"
      }
  instance-availability.rules: |
    ALERT InstanceDown
      IF up == 0
      FOR 1m
      LABELS { severity = "page" }
      ANNOTATIONS {
        summary = "Instance {{ $labels.instance }} down",
        description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.",
      }
  low-disk-space.rules: |
    ALERT NodeLowRootDisk
      IF ((node_filesystem_size{mountpoint="/root-disk"} - node_filesystem_free{mountpoint="/root-disk"} ) / node_filesystem_size{mountpoint="/root-disk"} * 100) > 75
      FOR 2m
      LABELS {
        severity="page"
      }
      ANNOTATIONS {
        SUMMARY = "{{$labels.instance}}: Low root disk space",
        DESCRIPTION = "{{$labels.instance}}: Root disk usage is above 75% (current value is: {{ $value }})"
      }

    ALERT NodeLowDataDisk
      IF ((node_filesystem_size{mountpoint="/data-disk"} - node_filesystem_free{mountpoint="/data-disk"} ) / node_filesystem_size{mountpoint="/data-disk"} * 100) > 75
      FOR 2m
      LABELS {
        severity="page"
      }
      ANNOTATIONS {
        SUMMARY = "{{$labels.instance}}: Low data disk space",
        DESCRIPTION = "{{$labels.instance}}: Data disk usage is above 75% (current value is: {{ $value }})"
      }
  mem-usage.rules: |
    ALERT NodeSwapUsage
      IF (((node_memory_SwapTotal-node_memory_SwapFree)/node_memory_SwapTotal)*100) > 75
      FOR 2m
      LABELS {
        severity="page"
      }
      ANNOTATIONS {
        SUMMARY = "{{$labels.instance}}: Swap usage detected",
        DESCRIPTION = "{{$labels.instance}}: Swap usage usage is above 75% (current value is: {{ $value }})"
      }

    ALERT NodeMemoryUsage
      IF (((node_memory_MemTotal-node_memory_MemFree-node_memory_Cached)/(node_memory_MemTotal)*100)) > 75
      FOR 2m
      LABELS {
        severity="page"
      }
      ANNOTATIONS {
        SUMMARY = "{{$labels.instance}}: High memory usage detected",
        DESCRIPTION = "{{$labels.instance}}: Memory usage is above 75% (current value is: {{ $value }})"
      }
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: prometheus-rules
  namespace: monitoring
```

### 创建prometheus-core
即创建prometheus-server服务。

prometheus-core-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus-core
  namespace: monitoring
  labels:
    app: prometheus
    component: core
spec:
  replicas: 1
  template:
    metadata:
      name: prometheus-main
      labels:
        app: prometheus
        component: core
    spec:
      serviceAccountName: prometheus-k8s
      containers:
      - name: prometheus
        image: prom/prometheus:v1.7.0
        args:
          - '-storage.local.retention=12h'
          - '-storage.local.memory-chunks=500000'
          - '-config.file=/etc/prometheus/prometheus.yaml'
          - '-alertmanager.url=http://alertmanager:9093/'
        ports:
        - name: webui
          containerPort: 9090
        resources:
          requests:
            cpu: 500m
            memory: 500M
          limits:
            cpu: 500m
            memory: 500M
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus
        - name: rules-volume
          mountPath: /etc/prometheus-rules
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-core
      - name: rules-volume
        configMap:
          name: prometheus-rules
```

创建对应service：
prometheus-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
    component: core
  annotations:
    prometheus.io/scrape: 'true'
spec:
  type: NodePort
  ports:
    - port: 9090
      protocol: TCP
      name: webui
  selector:
    app: prometheus
    component: core
```


如果以上pod都运行正常，那么prometheus大致安装完成了，接下来查看prometheus service对应的nodeport，然后就可以通过浏览器访问该service了。

![prometheus-pic](http://7xrnwq.com1.z0.glb.clouddn.com/20171112-proemeus3.jpg)

点击“status”中的target，可以看到所有的数据来源状态：

![prometueus-pic2](http://7xrnwq.com1.z0.glb.clouddn.com/201711121-prometueus-target.jpg)

## Grafana安装部署

Grafana的具体安装部署过程就不再赘述了，比较简单，之前的博客中有详细教程，这里只说一下与prometueus对接的配置。
其实也比较简单，就是一个添加数据源的操作。
进入grafana界面，点击datasource，进入添加datasource的界面。
![grafana-data-source](http://7xrnwq.com1.z0.glb.clouddn.com/20171112-datasource.jpg)

然后添加prometueus暴露出的cluster-ip:port即可，也可以用service名称代替，不过要加上namespace，比如：http://prometheus.monitoring:9090 。测试成功即可。



## 参考文章


[prometheus overview](https://prometheus.io/docs/introduction/overview/)

[Prometheus入门](https://www.hi-linux.com/posts/25047.html)

[Kubernetes 1.6 部署prometheus和grafana](http://www.tk4479.net/wenwst/article/details/76624019)

[基于Prometheus做多维度的容器监控](http://blog.yiyun.pro/%E5%9F%BA%E4%BA%8Eprometheus%E5%81%9A%E5%A4%9A%E7%BB%B4%E5%BA%A6%E7%9A%84%E5%AE%B9%E5%99%A8%E7%9B%91%E6%8E%A7/)



***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***