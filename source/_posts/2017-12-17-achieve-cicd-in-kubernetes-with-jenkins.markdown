---
layout: post
title:  "Kuberbetes-- 利用Jenkins在Kubernetes中实践CI/CD"
date:   2017-12-17 20:20:11
tags: 
  - Kubernetes
  - Jenkins
---


## 概述

本文利用jenkins在k8s中简单实践了一下CI/CD，部分实验内容来自[Set Up a CI/CD Pipeline with Kubernetes ](https://www.linux.com/blog/learn/chapter/Intro-to-Kubernetes/2017/5/set-cicd-pipeline-kubernetes-part-1-overview)，除此外，还试验了一把利用jenkins kubernetes plugin实现动态分配资源构建。

## 在kubernetes中简单实践jenkins

首先简单介绍下jenkins,jenkins是一个java编写的开源的持续集成工具。具体来说，他可以将软件构建，测试，发布等一系列流程自动化，达到一键部署的目的。在进行本实验前，首先要有一个k8s环境，这里不再赘述。

### 部署jenkins

这里存储用的是ceph rbd，所以先创建一个PVC：

jenkins-pvc.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
 name: jenkins-pvc
 namespace: cicd
spec:
 accessModes:
    - ReadWriteOnce
 resources:
   requests:
     storage: 20Gi
 storageClassName: ceph-web
```

部署jenkins:

jenkins-deployment.yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: cicd
  labels:
    app: jenkins
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: jenkins
        tier: jenkins
    spec:
      containers:
      - image: chadmoon/jenkins-docker-kubectl:latest
        name: jenkins
        securityContext:
          privileged: true
        ports:
        - containerPort: 8080
          name: jenkins
        - containerPort: 50000
          name: agent
          protocol: TCP
        volumeMounts:
        - name: docker
          mountPath: /var/run/docker.sock
        - name: jenkins-persistent-storage
          mountPath: /root/.jenkins
        - name: kube-config
          mountPath: /root/.kube/config
        - name: image-registry
          mountPath: /root/.docker
      volumes:
      - name: docker
        hostPath:
          path: /var/run/docker.sock
      - name: jenkins-persistent-storage
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: kube-config
        hostPath:
          path: /root/.kube/config
      - name: image-registry
        configMap:
          name: image-registry-auth
```
简单解释一下：
- 该镜像除了安装jenkins，还装了docker cli（与host docker daemon交互），kubectl（与k8s apiserver交互）
- 容器开了两个端口，一个用于web-ui,一个用于后面实验jenkins kubernetes plugin时与JNLP slave agents 交互。
- 挂载了四个volume，依次是，一个用于docker cli，一个用于存储jenkins数据，一个用于kubectl与k8s交互验证，最后挂载了一个configmap，与image registry（我们用的harbor）交互验证。 

部署jenkins service & ingress:

jenkins-service-ingress.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-web-ui
  namespace: cicd
  labels:
    app: jenkins
spec:
  ports:
    - port: 80
      targetPort: 8080
      name: web-ui
    - port: 50000
      targetPort: 50000
      name: agent
  selector:
    app: jenkins
    tier: jenkins
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-web-ui
  namespace: cicd
spec:
  rules:
  - host: jenkins.com
    http:
      paths:
      - backend:
          serviceName: jenkins-web-ui
          servicePort: 80
```

完成上述所有操作后，查看一下对应的pod 是否running。

### 配置pipeline

按照ingress的配置，修改本地host，然后浏览器中输入http://jenkins.com ，就进入到jenkins 的web-ui了。

按照提示创建完用户后，开始进入CICD 的实验环节：

新建一个item，命名并选中pipeline:

![new-item](http://7xrnwq.com1.z0.glb.clouddn.com/20171218184207-new-item.jpg)

pipeline 配置如下：

![pipeline](http://7xrnwq.com1.z0.glb.clouddn.com/2017121818480-pipeline.jpg)

在Git Repository URL部分添加github url,这里用的是[我的github-test-kubernetes-ci-cd](https://github.com/zhangchenchen/kubernetes-ci-cd)，是直接fork自[kubernetes-ci-cd](https://github.com/kenzanlabs/kubernetes-ci-cd)，并做了一些更改,之后保存就可以了。
进入刚创建的item，点击立即构建：

![build](http://7xrnwq.com1.z0.glb.clouddn.com/20171218185344-build.jpg)

之后就可以看到构建信息了，如果出错也可以查看对应步骤的log。
同时，我们的应用也已经部署到k8s中了。
![test-jenkins](http://7xrnwq.com1.z0.glb.clouddn.com/20171218190117-test-jenkins.jpg)

### 步骤详解

接下来看一下点击“立即构建”后发生了什么，点击后，jenkins首先是从github检出项目代码，然后根据检出的项目中根目录下的Jenkinsfile进行项目构建，看下该项目的Jenkinsfile。
```yaml
node {

    checkout scm

    env.DOCKER_API_VERSION="1.23"
    
    sh "git rev-parse --short HEAD > commit-id"

    tag = readFile('commit-id').replace("\n", "").replace("\r", "")
    appName = "hello-kenzan"
    registryHost = "172.16.21.253:10080/library/"
    imageName = "${registryHost}${appName}:${tag}"
    env.BUILDIMG=imageName

    stage "Build"
    
        sh "docker build -t ${imageName} -f applications/hello-kenzan/Dockerfile applications/hello-kenzan"
    
    stage "Push"

        sh "docker push ${imageName}"

    stage "Deploy"

        sh "sed 's#127.0.0.1:30400/hello-kenzan:latest#'$BUILDIMG'#' applications/hello-kenzan/k8s/deployment.yaml | kubectl apply -f -"
        sh "kubectl rollout status deployment/hello-kenzan"
}
```

可以看到，Jenkinsfile定义了三个阶段，第一个阶段是“Build”,这个阶段是根据给定的Dockerfile创建一个镜像，第二个阶段“Push”,把生成的镜像push到我们的镜像仓库中，最后一个阶段是"Deploy"，编辑了一下deployment.yaml模板，然后调用kubectl命令进行部署。
看一下“Build”阶段的dockerfile:
```yaml
FROM nginx:latest

COPY index.html /usr/share/nginx/html/index.html
COPY DockerFileEx.jpg /usr/share/nginx/html/DockerFileEx.jpg

EXPOSE 80
```
就是一个很简单的nginx应用。
再看下deployment.yaml：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-kenzan
  labels:
    app: hello-kenzan
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: hello-kenzan
    tier: hello-kenzan
  type: NodePort

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-kenzan
  labels:
    app: hello-kenzan
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: hello-kenzan
        tier: hello-kenzan
    spec:
      containers:
      - image: 127.0.0.1:30400/hello-kenzan:latest
        name: hello-kenzan
        ports:
        - containerPort: 80
          name: hello-kenzan
```

这里服务发现使用的nodeport，可以改为其他方式（比如ingress）。

接下来，可以试着修改一下index.html，然后push到github中，再构建一下，看看内容是否改变。

## 利用jenkins kubernetes plugin实现动态分配资源构建

在上述实例中，我们利用jenkins实现了一个小应用的CI/CD，这个小应用非常简单，“build”阶段就是直接在本地调用host docker构建的镜像，设想一下，如果这个应用需要编译，需要测试，那么这个时间就长了，而且如果都在本地构建的话，一个人使用还好，如果多个人一起构建，就会造成拥塞。
为了解决上述问题，我们可以充分利用k8s的容器编排功能，jenkins接收到任务后，调用k8s api，创造新的 agent pod，将任务分发给这些agent pod，agent pod执行任务，任务完成后将结果汇总给jenkins pod，同时删除完成任务的agent pod。
为了实现上述功能，我们需要给jenkins安装一个插件，叫做[jenkins kubernetes plugin](https://github.com/carlossg/jenkins-kubernetes-plugin)。

### 插件安装与配置

安装比较简单，直接到jenkins 界面的系统管理，插件管理界面进行安装就可以了。
安装好之后，进入系统管理----->系统设置，最下面有一个“云”，选择“新增一个云”---->kubernetes。

![jenkins-k8s](http://7xrnwq.com1.z0.glb.clouddn.com/20171218194329-jenkins-k8s.jpg)
这里没有配置k8s，因为如果不配置api-server的话，jenkins会默认使用~/.kube/config下的配置，而我们已经在~/.kube/config做过配置了，所以这里就不做了。Jenkins URL我们使用的是集群内的服务地址。
再往下看 kubernetes pod template配置：

![jenkins-pod-template](http://7xrnwq.com1.z0.glb.clouddn.com/20171218195151jenkins-pod-template.jpg)

这个pod tempalte就是之后我们创建 agent使用的模板，镜像使用“jenkins/jnlp-slave:alpine”，配置完成后，点击保存。
然后还要配置一下agent与jenkins通信的端口，在系统管理---->Configure Global Security，指定端口为我们之前设定的5000端口：

![jenkins-port](http://7xrnwq.com1.z0.glb.clouddn.com/20171218195724-jenkins-port.jpg)

### 简单测试

配置完成后做一个简单的测试。

新建一个item，这里选择“构建一个自由风格的软件项目”：

![test-jnlp](http://7xrnwq.com1.z0.glb.clouddn.com/20171218200215test-jnlp.jpg)

配置时注意在General部分有一个restrict：

![jenkins-general](http://7xrnwq.com1.z0.glb.clouddn.com/20171218200511-general.jpg)
Label Expression就写之前我们k8s podtemplate 的label。

在构建部分我们写一个简单的测试命令：echo TRUE

![jenkins-build](http://7xrnwq.com1.z0.glb.clouddn.com/20171218200831-jenkins-build.jpg)

点击立即构建，如果成功的话，我们在“管理主机”模块会看到新增了一个主机：

![add-host](http://7xrnwq.com1.z0.glb.clouddn.com/20171218162939-multi-host.jpg)
同时，也会在k8s中发现新创建了一个名为jnlp-slave-8bq5m的pod。
任务结束后，pod删除，主机消失，在console output 会看到执行结果：

![console-optput](http://7xrnwq.com1.z0.glb.clouddn.com/20171218201618-console-output.jpg)


## 出现问题总结

### jnlp-slave pod创建失败

查看pod日志，发现是连接不上jenkins ，通过修改Configure Global Security的启用安全，TCP port for JNLP agents 
指定端口解决。

### jnlp-slave pod 无法删除

因为我们执行构建后，如果 jnlp-slave pod创建失败，它会不断的尝试创建新的pod，并试图连接jenkins，一段时间后，就会创造很多失败的jnlp-slave pod。如果遇到这种情况，需要尽早中断任务并删除失败的pod。
在删除某个pod时 ，该pod一直处于termating阶段，kubectl delete无法删除。后来参考[Pods stuck at terminated status](https://stackoverflow.com/questions/35453792/pods-stuck-at-terminated-status)，使用如下命令解决：
```bash
kubectl delete pod NAME --grace-period=0 --force
```


## 参考文章

[Set Up a CI/CD Pipeline with a Jenkins Pod in Kubernetes ](https://www.linux.com/blog/learn/chapter/Intro-to-Kubernetes/2017/6/set-cicd-pipeline-jenkins-pod-kubernetes-part-2)

[Achieving CI/CD with Kubernetes](http://blog.sonatype.com/achieving-ci/cd-with-kubernetes)

[容器时代CI/CD平台中的Kubernetes调度器定制方法](http://blog.csdn.net/WaltonWang/article/details/73477812)

[Jenkinsfile使用](https://www.w3cschool.cn/jenkins/jenkins-qc8a28op.html)

[安装和设置 kubectl](http://kubernetes.kansea.com/docs/user-guide/prereqs/)

[jenkins-kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin)

[基于Kubernetes 部署 jenkins 并动态分配资源](http://www.cnblogs.com/hahp/p/5812455.html)

[使用Kubernetes-Jenkins实现CI/CD](https://www.kubernetes.org.cn/1791.html)

***本篇文章由[pekingzcc](https://zhangchenchen.github.io/)采用[知识共享署名-非商业性使用 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)进行许可,转载请注明。***


 ***END***