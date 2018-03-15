---
layout: post
title: kubernetes中部署spark集群
description: 在kubernetes中部署spark集群
date: 2018.03.02 21:00 +08:00
tags: 
- kubernetes
- spark
---

在写这个的时候，spark版本为2.2.1。

#### 基于kubernetes部署的两种方式

* 直接使用kubernetes作为集群管理器(Cluster Manager)，类似与mesos和yarn，使用方式可以看[running-on-kubernetes](https://apache-spark-on-k8s.github.io/userdocs/running-on-kubernetes.html)。但是这个部署方式，一是还不成熟，不推荐在生产环境使用。第二是要求k8s版本大于1.6，但我这边版本1.5.1，线上在用，不太想升级，而spark只是想搭起来玩玩...
* 第二种方式是standalone的方式，即便是不用集群也能很方便的用sbin下的脚本来部署，不过使用k8s有几个好处，一个是提高机器使用率。这边的k8s集群大部分是在白天使用，晚上空闲，刚好能拿来跑数据。二是方便一键扩容，一键升级，能复用本身在k8s集群上做好的监控以及日志收集

#### 在k8s上部署standalone集群

以下内容主要依据[github 上k8s example中spark的例子](https://github.com/kubernetes/examples/blob/master/staging/spark/README.md)。做了一些适应版本的修改。

首先我们需要一个有spark以及其依赖的的docker镜像，这边我就很简单的下载了[spark-2.2.1-bin-hadoop2.7](https://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-2.2.1/spark-2.2.1-bin-hadoop2.7.tgz)并放在了/opt目录下，打包的dockerfile就不发了，docker store上也有很多做好的镜像。另外，k8s需要有dns（现在似乎默认带的）。

##### namespace

为了方便管理，还是新建一个namespace，将所有的相关的都放在这个namespace下，方便资源管理。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "spark-cluster"
  labels:
    name: "spark-cluster"
```

kubectl create -f namespace-spark-cluster.yaml  新建一个名为spark-cluster的namespace。

##### master

master分为两个部分，一个是类型为rc的主体，命名为spark-master-controller.yaml，另一部分为一个service，暴露master的端口给slaver使用。

```yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-master-controller
spec:
  replicas: 1
  selector:
    component: spk-master
  template:
    metadata:
      labels:
        component: spk-master
    spec:
      containers:
        - name: spk-master
          image: spark:2.2.1.1
          command: ["/bin/sh"]
          args: ["-c","sh /opt/spark-2.2.0-bin-hadoop2.7/sbin/start-master.sh && tail -f /opt/spark-2.2.0-bin-hadoop2.7/logs/spark--org.apache.spark.deploy.master.Master-1-*"]
          ports:
            - containerPort: 7077
            - containerPort: 8080
```

以上为controller，直接使用spark的start-master脚本启动，但是启动后他会退到后台，导致k8s启动不了pod，所以还加了个tail -f一个master输出的log，顺便也方便查看log。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: spk-master
spec:
  ports:
    - port: 7077
      targetPort: 7077
      name: spark
    - port: 8080
      targetPort: 8080
      name: http
  selector:
    component: spk-master
```

一个service，把7077端口和8080端口暴露出来给集群，方便slaver直接用spk-master:8080这样的方式进行访问。注意，只是暴露给集群，外部访问的方式最后会说。

kubectl create -f spark-master-controller.yaml —namespace=spark-cluster

kubectl create -f spark-master-service.yaml —namespace=spark-cluster

**这里有个坑，start-master这个启动脚本中会用到SPARK_MASTER_PORT这个参数，而上边这个service如果名字为spark-master的话刚好冲突了，会把SPARK_MASTER_PORT设置为 host:port的形式，导致脚本启动失败。所以我一股脑把所有的spark-master改成spk-master了**

##### worker

```yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-worker-controller
spec:
  replicas: 1
  selector:
    component: spark-worker
  template:
    metadata:
      labels:
        component: spark-worker
    spec:
      containers:
        - name: spark-worker
          image: spark:2.2.1.1
          command: ["/bin/sh"]
          args: ["-c","sh /opt/spark-2.2.0-bin-hadoop2.7/sbin/start-slave.sh spark://spk-master.spark-cluster:7077;tail -f /opt/spark-2.2.0-bin-hadoop2.7/logs/spark--org.apache.spark.deploy.worker.Worker*"]
          ports:
            - containerPort: 8081
          resources:
            requests:
              cpu: "1"
              memory: "10Gi"
            limits:
              cpu: "2"
              memory: "12Gi"
```

kubectl create -f spark-worker-controller.yaml —namespace=spark-cluster

启动worker脚本中需要传入master的地址，因为有dns且设置了service的缘故，可以通过spk-master.spark-cluster访问。把replicas设置为N即可启动N个worker。另外，我还在worker上加了资源的限制，限制最多使用2个cpu以及12g内存。

##### proxy

image为elsonrodriguez/spark-ui-proxy:1.0 这玩意在一般启动standalone集群的时候是没有的，但是在k8s集群里边，则必不可缺。

设想一下，如果只是简单的暴露master的8080端口出来，我们只能看到master的管理页面，但是进一步从master访问worker的ui则变得不太现实（每个worker都有自己的ui地址，且ip分配很随机，这些ip只能在集群内部访问）。所以我们需要一个代理服务，从内部访问完我们需要的页面后，返回给我们，这样我们只需要暴露一个代理的地址即可。

```yaml
kind: ReplicationController
apiVersion: v1
metadata:
  name: spark-ui-proxy-controller
spec:
  replicas: 1
  selector:
    component: spark-ui-proxy
  template:
    metadata:
      labels:
        component: spark-ui-proxy
    spec:
      containers:
        - name: spark-ui-proxy
          image: elsonrodriguez/spark-ui-proxy:1.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
          args:
            - spk-master:8080
          livenessProbe:
              httpGet:
                path: /
                port: 80
              initialDelaySeconds: 120
              timeoutSeconds: 5
```

kubectl create -f spark-ui-proxy-controller.yaml —namespace=spark-cluster

并且暴露proxy的80端口

```yaml
kind: Service
apiVersion: v1
metadata:
  name: spark-ui-proxy
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32180
  selector:
    component: spark-ui-proxy
```

kubectl create -f spark-ui-proxy-service.yaml —namespace=spark-cluster

至此，集群搭建完毕。可以通过集群的32180端口访问管理页面。










​				
​			
​		
​	