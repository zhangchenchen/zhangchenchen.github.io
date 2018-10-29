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

不过Prometheus也不是万能的，他不适于一些实时要求比较高的监控项目，且因为是单机存储，存储量有限，局限了存储时间。prometheus集成了leveldb，一个能高效插入数据的数据库，在ssd盘下io占用比较高，可能会有大量数据堆积内存，内存占用率大（可以进行适当配置）。

Prometheus的整体架构如下：

![prometheus-arch](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/201711-12-prometheous-arch.jpg)

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

![prometheus-pic](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20171112-proemeus3.jpg)

点击“status”中的target，可以看到所有的数据来源状态：

![prometueus-pic2](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/201711121-prometueus-target.jpg)

## Grafana安装部署

Grafana的具体安装部署过程就不再赘述了，比较简单，之前的博客中有详细教程，这里只说一下与prometueus对接的配置。
其实也比较简单，就是一个添加数据源的操作。
进入grafana界面，点击datasource，进入添加datasource的界面。
![grafana-data-source](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20171112-datasource.jpg)

然后添加prometueus暴露出的cluster-ip:port即可，也可以用service名称代替，不过要加上namespace，比如：http://prometheus.monitoring:9090 。测试成功即可。

## alertmanager 安装部署

以上步骤完成后，还需要一个报警机制，alertmanager作为一个prometueus的独立模块实现了报警机制，安装如下：

### 创建alertmanager 配置文件

本示例中只使用了邮箱报警（注意邮箱的配置），也可以使用slack报警配置，如果考虑其他比如短信或微信报警可以与onealert进行集成。
alert-manager-configmap.yaml
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
data:
  config.yml: |-
    global:
      # ResolveTimeout is the time after which an alert is declared resolved
      # if it has not been updated.
      resolve_timeout: 5m
      # The smarthost and SMTP sender used for mail notifications.
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'foo@bar.com'
      smtp_auth_username: 'foo@bar.com'
      smtp_auth_password: 'barfoo'
      # The API URL to use for Slack notifications.
      #slack_api_url: 'https://hooks.slack.com/services/abc123'
      # # The auth token for Hipchat.
      # hipchat_auth_token: '1234556789'
      #
      # # Alternative host for Hipchat.
      # hipchat_url: 'https://hipchat.foobar.org/'
    # # The directory from which notification templates are read.
    templates:
    - '/etc/alertmanager-templates/*.tmpl'
    # The root route on which each incoming alert enters.
    route:
      # The labels by which incoming alerts are grouped together. For example,
      # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
      # be batched into a single group.
      group_by: ['alertname', 'cluster', 'service']
      # When a new group of alerts is created by an incoming alert, wait at
      # least 'group_wait' to send the initial notification.
      # This way ensures that you get multiple alerts for the same group that start
      # firing shortly after another are batched together on the first
      # notification.
      group_wait: 30s
      # When the first notification was sent, wait 'group_interval' to send a batch
      # of new alerts that started firing for that group.
      group_interval: 5m
      # If an alert has successfully been sent, wait 'repeat_interval' to
      # resend them.
      #repeat_interval: 1m
      repeat_interval: 15m
      # A default receiver
      # If an alert isn't caught by a route, send it to default.
      receiver: default
      # All the above attributes are inherited by all child routes and can
      # overwritten on each.
      # The child route trees.
      routes:
      # Send severity=slack alerts to slack.
      - match:
          severity: slack
        receiver: slack_alert
      - match:
          severity: email
        receiver: slack_alert
    #   receiver: email_alert
    receivers:
    - name: 'default'
      email_configs:
      - to: 'foo@bar.com'
```

### 创建报警格式配置文件

alertmanager-templates.yaml
```yaml
apiVersion: v1
data:
  default.tmpl: |
    {{ define "__alertmanager" }}AlertManager{{ end }}
    {{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}
    {{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
    {{ define "__description" }}{{ end }}
    {{ define "__text_alert_list" }}{{ range . }}Labels:
    {{ range .Labels.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Annotations:
    {{ range .Annotations.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Source: {{ .GeneratorURL }}
    {{ end }}{{ end }}
    {{ define "slack.default.title" }}{{ template "__subject" . }}{{ end }}
    {{ define "slack.default.username" }}{{ template "__alertmanager" . }}{{ end }}
    {{ define "slack.default.fallback" }}{{ template "slack.default.title" . }} | {{ template "slack.default.titlelink" . }}{{ end }}
    {{ define "slack.default.pretext" }}{{ end }}
    {{ define "slack.default.titlelink" }}{{ template "__alertmanagerURL" . }}{{ end }}
    {{ define "slack.default.iconemoji" }}{{ end }}
    {{ define "slack.default.iconurl" }}{{ end }}
    {{ define "slack.default.text" }}{{ end }}
    {{ define "hipchat.default.from" }}{{ template "__alertmanager" . }}{{ end }}
    {{ define "hipchat.default.message" }}{{ template "__subject" . }}{{ end }}
    {{ define "pagerduty.default.description" }}{{ template "__subject" . }}{{ end }}
    {{ define "pagerduty.default.client" }}{{ template "__alertmanager" . }}{{ end }}
    {{ define "pagerduty.default.clientURL" }}{{ template "__alertmanagerURL" . }}{{ end }}
    {{ define "pagerduty.default.instances" }}{{ template "__text_alert_list" . }}{{ end }}
    {{ define "opsgenie.default.message" }}{{ template "__subject" . }}{{ end }}
    {{ define "opsgenie.default.description" }}{{ .CommonAnnotations.SortedPairs.Values | join " " }}
    {{ if gt (len .Alerts.Firing) 0 -}}
    Alerts Firing:
    {{ template "__text_alert_list" .Alerts.Firing }}
    {{- end }}
    {{ if gt (len .Alerts.Resolved) 0 -}}
    Alerts Resolved:
    {{ template "__text_alert_list" .Alerts.Resolved }}
    {{- end }}
    {{- end }}
    {{ define "opsgenie.default.source" }}{{ template "__alertmanagerURL" . }}{{ end }}
    {{ define "victorops.default.message" }}{{ template "__subject" . }} | {{ template "__alertmanagerURL" . }}{{ end }}
    {{ define "victorops.default.from" }}{{ template "__alertmanager" . }}{{ end }}
    {{ define "email.default.subject" }}{{ template "__subject" . }}{{ end }}
    {{ define "email.default.html" }}
    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
    <!--
    Style and HTML derived from https://github.com/mailgun/transactional-email-templates
    The MIT License (MIT)
    Copyright (c) 2014 Mailgun
    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
    -->
    <html xmlns="http://www.w3.org/1999/xhtml" xmlns="http://www.w3.org/1999/xhtml" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
    <head style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
    <meta name="viewport" content="width=device-width" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
    <title style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">{{ template "__subject" . }}</title>
    </head>
    <body itemscope="" itemtype="http://schema.org/EmailMessage" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; -webkit-font-smoothing: antialiased; -webkit-text-size-adjust: none; height: 100%; line-height: 1.6em; width: 100% !important; background-color: #f6f6f6; margin: 0; padding: 0;" bgcolor="#f6f6f6">
    <table style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; width: 100%; background-color: #f6f6f6; margin: 0;" bgcolor="#f6f6f6">
      <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
        <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0;" valign="top"></td>
        <td width="600" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; display: block !important; max-width: 600px !important; clear: both !important; width: 100% !important; margin: 0 auto; padding: 0;" valign="top">
          <div style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; max-width: 600px; display: block; margin: 0 auto; padding: 0;">
            <table width="100%" cellpadding="0" cellspacing="0" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; border-radius: 3px; background-color: #fff; margin: 0; border: 1px solid #e9e9e9;" bgcolor="#fff">
              <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 16px; vertical-align: top; color: #fff; font-weight: 500; text-align: center; border-radius: 3px 3px 0 0; background-color: #E6522C; margin: 0; padding: 20px;" align="center" bgcolor="#E6522C" valign="top">
                  {{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }} for {{ range .GroupLabels.SortedPairs }}
                    {{ .Name }}={{ .Value }}
                  {{ end }}
                </td>
              </tr>
              <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 10px;" valign="top">
                  <table width="100%" cellpadding="0" cellspacing="0" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <a href="{{ template "__alertmanagerURL" . }}" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; color: #FFF; text-decoration: none; line-height: 2em; font-weight: bold; text-align: center; cursor: pointer; display: inline-block; border-radius: 5px; text-transform: capitalize; background-color: #348eda; margin: 0; border-color: #348eda; border-style: solid; border-width: 10px 20px;">View in {{ template "__alertmanager" . }}</a>
                      </td>
                    </tr>
                    {{ if gt (len .Alerts.Firing) 0 }}
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">[{{ .Alerts.Firing | len }}] Firing</strong>
                      </td>
                    </tr>
                    {{ end }}
                    {{ range .Alerts.Firing }}
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">Labels</strong><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                        {{ range .Labels.SortedPairs }}{{ .Name }} = {{ .Value }}<br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        {{ if gt (len .Annotations) 0 }}<strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">Annotations</strong><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        {{ range .Annotations.SortedPairs }}{{ .Name }} = {{ .Value }}<br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        <a href="{{ .GeneratorURL }}" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; color: #348eda; text-decoration: underline; margin: 0;">Source</a><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                      </td>
                    </tr>
                    {{ end }}
                    {{ if gt (len .Alerts.Resolved) 0 }}
                      {{ if gt (len .Alerts.Firing) 0 }}
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                        <hr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                        <br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                      </td>
                    </tr>
                      {{ end }}
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">[{{ .Alerts.Resolved | len }}] Resolved</strong>
                      </td>
                    </tr>
                    {{ end }}
                    {{ range .Alerts.Resolved }}
                    <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                      <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0; padding: 0 0 20px;" valign="top">
                        <strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">Labels</strong><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                        {{ range .Labels.SortedPairs }}{{ .Name }} = {{ .Value }}<br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        {{ if gt (len .Annotations) 0 }}<strong style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">Annotations</strong><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        {{ range .Annotations.SortedPairs }}{{ .Name }} = {{ .Value }}<br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />{{ end }}
                        <a href="{{ .GeneratorURL }}" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; color: #348eda; text-decoration: underline; margin: 0;">Source</a><br style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;" />
                      </td>
                    </tr>
                    {{ end }}
                  </table>
                </td>
              </tr>
            </table>
            <div style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; width: 100%; clear: both; color: #999; margin: 0; padding: 20px;">
              <table width="100%" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                <tr style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; margin: 0;">
                  <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 12px; vertical-align: top; text-align: center; color: #999; margin: 0; padding: 0 0 20px;" align="center" valign="top"><a href="{{ .ExternalURL }}" style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 12px; color: #999; text-decoration: underline; margin: 0;">Sent by {{ template "__alertmanager" . }}</a></td>
                </tr>
              </table>
            </div></div>
        </td>
        <td style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; box-sizing: border-box; font-size: 14px; vertical-align: top; margin: 0;" valign="top"></td>
      </tr>
    </table>
    </body>
    </html>
    {{ end }}
    {{ define "pushover.default.title" }}{{ template "__subject" . }}{{ end }}
    {{ define "pushover.default.message" }}{{ .CommonAnnotations.SortedPairs.Values | join " " }}
    {{ if gt (len .Alerts.Firing) 0 }}
    Alerts Firing:
    {{ template "__text_alert_list" .Alerts.Firing }}
    {{ end }}
    {{ if gt (len .Alerts.Resolved) 0 }}
    Alerts Resolved:
    {{ template "__text_alert_list" .Alerts.Resolved }}
    {{ end }}
    {{ end }}
    {{ define "pushover.default.url" }}{{ template "__alertmanagerURL" . }}{{ end }}
  slack.tmpl: |
    {{ define "slack.devops.text" }}
    {{range .Alerts}}{{.Annotations.DESCRIPTION}}
    {{end}}
    {{ end }}
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: alertmanager-templates
  namespace: monitoring
```

### 创建 alertmanager 应用

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      name: alertmanager
      labels:
        app: alertmanager
    spec:
      containers:
      - name: alertmanager
        image: quay.io/prometheus/alertmanager:v0.7.1
        args:
          - '-config.file=/etc/alertmanager/config.yml'
          - '-storage.path=/alertmanager'
        ports:
        - name: alertmanager
          containerPort: 9093
        volumeMounts:
        - name: config-volume
          mountPath: /etc/alertmanager
        - name: templates-volume
          mountPath: /etc/alertmanager-templates
        - name: alertmanager
          mountPath: /alertmanager
      volumes:
      - name: config-volume
        configMap:
          name: alertmanager
      - name: templates-volume
        configMap:
          name: alertmanager-templates
      - name: alertmanager
        emptyDir: {}
```


### 创建 alertmanager service

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
    prometheus.io/path: '/metrics'
  labels:
    name: alertmanager
  name: alertmanager
  namespace: monitoring
spec:
  selector:
    app: alertmanager
  type: NodePort
  ports:
  - name: alertmanager
    protocol: TCP
    port: 9093
    targetPort: 9093
```

创建完成后，可以通过alertmanager service的nodeport进入alertmanager 的管理界面，里面有一些设置选项，如下：
![alert-manager](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/QQ%E6%88%AA%E5%9B%BE20171114140712.jpg)

具体设置可以参考[Prometheus监控 - Alertmanager报警模块](https://sagittariusyx.github.io/2016/03/07/prometheus-alertmanager/)。
可以通过stop 某个节点的kubelet程序的方式测试一下报警模块是否运行正常。

这样，所有的工作基本完成，接下来就是根据自己的需求进行监控的配置调整，比如grafana面板的管理，报警的具体要求，阈值等。

## ------- 更新 ------2017-11-23-----增加ceph监控

我司容器云平台后端存储使用的ceph,花了一点时间将ceph的监控转到prometheus上，并找了一个不错的grafana模板，最终效果（实验环境）如下：

![grafana-cepg-cluster](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20171123-grafana-ceph-cluster.jpg)

![grafana-ceph-osd](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20171123-ceph-osd.jpg)

简单叙述下具体过程：

### 部署 ceph-exporter

[ceph-exporter](https://github.com/digitalocean/ceph_exporter)是digitalocean开源的一个获取ceph metric的工具，可以直接在某一台物理机上部署，也可以pod形式部署。我们这里使用pod形式部署，yaml文件如下：

ceph-exporter-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ceph-exporter
  namespace: monitoring
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: ceph-exporter
    spec:
      containers:
        - name: ceph-exporter
          image: digitalocean/ceph_exporter
          imagePullPolicy: IfNotPresent
          volumeMounts:
          - mountPath: /etc/ceph
            name: ceph-conf
      volumes:
      - hostPath:
          path: /etc/ceph
        name: ceph-conf
```

创建service：

ceph-exporter-svc.yaml
```yaml
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    app: ceph-exporter
    k8s-app: ceph-exporter
  name: ceph-exporter
  namespace: monitoring
spec:
  ports:
  - name: ceph-exporter
    port: 9128
    protocol: TCP
    targetPort: 9128
  selector:
    app: ceph-exporter
```

注意：service配置文件中prometheus.io/scrape: 'true' ，表示该service可以被prometheus 抓取到。

部署完成后，到prometheus 的target页面查看是否能被prometheus 抓取。

![prometheus-target](https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20171123-ceph-prometheus.jpg)


### 导入 grafana 模板

grafana 模板可以直接在[官网](https://grafana.com/dashboards)搜索。这里的模板使用的[这个](https://github.com/SUSE/grafana-dashboards-ceph)，直接导入对应json文件，注意要将"datasource"改为自己的datasource name,不然会报错。


## 参考文章

[kubernetes-prometheus](https://github.com/giantswarm/kubernetes-prometheus)

[prometheus overview](https://prometheus.io/docs/introduction/overview/)

[Prometheus入门](https://www.hi-linux.com/posts/25047.html)

[Kubernetes 1.6 部署prometheus和grafana](http://www.tk4479.net/wenwst/article/details/76624019)

[基于Prometheus做多维度的容器监控](http://blog.yiyun.pro/%E5%9F%BA%E4%BA%8Eprometheus%E5%81%9A%E5%A4%9A%E7%BB%B4%E5%BA%A6%E7%9A%84%E5%AE%B9%E5%99%A8%E7%9B%91%E6%8E%A7/)

[用K8S+Prometheus四步搭建Ceph监控](http://www.jianshu.com/p/0dcdbc1135bd)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***