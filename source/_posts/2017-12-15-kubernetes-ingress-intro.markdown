---
layout: post
title:  "Kuberbetes-- kubernetes ingress 实践"
date:   2017-12-15 19:08:12
tags: 
  - Kubernetes
---


## 概述

k8s集群暴露内部服务有多种方式：
- Nodeport Service: 通过host port做端口映射，由kube-proxy实现。
- LoadBalancer Service: 利用云平台提供的LoadBalancer服务，也是由kube-proxy实现。
- Ingress: k8s1.2版本新增功能，由反向代理负载均衡器，Ingress Controller，ingress配置共同实现。
- Port Proxy: 起一个pod专门做端口转发，可以将pod/host port 转发给k8s service，参考[Proxy a pod port or host port to a kubernetes Service](https://github.com/kubernetes/contrib/tree/master/for-demos/proxy-to-service)
- Service loadbalancer: 在物理机上部署一套Service loadbalancer，参考[Bare Metal Service Load Balancers](https://github.com/kubernetes/contrib/tree/master/service-loadbalancer)

其中，后两种方法因为相对比较复杂，逐渐被遗弃。目前用的比较多的就是前三种,LoadBalancer Service需要云服务的参与，NodePort 方式在集群内服务数量可控的情况下可以使用，当服务数量增多时需要注意端口管理，防止端口冲突。Ingress方式需要我们自行部署ingress controller服务，本篇文章详述一下Ingress方式。

## Ingress 原理

在讲Ingres实现原理时，我们可以从NodePort入手，NodePort最大的弊端就是端口多了之后难以管理，我们自然而然的就会想到利用反向代理工具（比如 nginx）只监听host上一个端口，然后再根据请求的域名转发给集群内部服务，这就要求这个nginx能够转发到集群内部，这个简单，我们直接将nginx部署到集群内部就可以了。
接下来的问题就是如何配置nginx了，这就要借助k8s中的ingress了，ingress实际上就是一个yaml文件，真正执行配置nginx的是叫做ingress controller的程序，它会调用k8s 的api，获取ingress，然后根据这个yaml文件生成nginx 配置模板，写入nginx。除此之外，Ingress Controller 通过不断地跟 kubernetes API 打交道，实时的感知后端 service、pod 等变化，比如新增和减少 pod，service 增加与减少等；当得到这些变化信息后，Ingress Controller 再结合 Ingress 生成配置，然后更新反向代理负载均衡器，并刷新其配置，达到服务发现的目的。

贴一张图，图片来自[这里](https://mritd.me/2016/12/06/try-traefik-on-kubernetes/#13ingress)

![ingress-arch](http://7xrnwq.com1.z0.glb.clouddn.com/20171214-ingress-arch.jpg)

## Ingress 实践

ingress包括反向代理负载均衡器，Ingress Controller，ingress三部分，通常反向代理负载均衡器与Ingress Controller会部署到同一个pod中，一个负责反向代理，一个负责与k8s交互并更新配置。不过随着微服务的流行，有人将这两个功能合在了一块，就是traefik。这样，大致结构就是这样：

![traffic](http://7xrnwq.com1.z0.glb.clouddn.com/20171214traffic.jpg)

接下来开始部署：

因为有RBAC，所以先部署相应角色：
traefik-rbac.yaml
```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

接下来开始部署traefik(可以使用deployment或DaemonSet):

traefik-daemonset.yaml
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
        securityContext:
          privileged: true
        args:
        - -d
        - --web
        - --kubernetes
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
  type: NodePort
```

创建一个ingress，将traffik的web ui 暴露出来：

traefik-ingress.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik-ui.minikube
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

Ingress spec 中包含配置一个loadbalancer或proxy server的所有信息。最重要的是，它包含了一个匹配所有入站请求的规则列表。目前ingress只支持http规则。每条http规则包含以下信息：一个host配置项（比如traefik-ui.minikube，默认是*），path列表（比如：/testpath，默认是:/），每个path都关联一个backend(比如traefik-web-ui:80)。在loadbalancer将流量转发到backend之前，所有的入站请求都要先匹配host和path。
如果没有rules，可以指定一个默认backend（例如404 page）。

部署完成，修改本地host，将traefik-ui.minikube 指向host ip。浏览器中输入http://traefik-ui.minikube/dashboard/#/即可：

![traffic-web-ui](http://7xrnwq.com1.z0.glb.clouddn.com/20171214-traffic-web-ui.jpg)

生产环境中可以利用dns再做一层负载均衡。

如果需要部署 https，可以参考[Using Traefik with TLS on Kubernete](https://medium.com/@patrickeasters/using-traefik-with-tls-on-kubernetes-cb67fb43a948)

## 参考文章

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

[Kubernetes Ingress Controller](https://docs.traefik.io/user-guide/kubernetes/)

[Traefik-kubernetes 初试](https://mritd.me/2016/12/06/try-traefik-on-kubernetes/)

[Kubernetes Nginx Ingress 教程](https://mritd.me/2017/03/04/how-to-use-nginx-ingress/)

[Kubernetes Ingress解析](https://www.kubernetes.org.cn/1885.html)


***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***