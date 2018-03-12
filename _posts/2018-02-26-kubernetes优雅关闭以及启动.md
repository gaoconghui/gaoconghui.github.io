---
layout: post
title: 优雅关闭以及机器kubernetes pods
description: 很常见的一个问题，一个服务需要优雅关闭以及优雅启动，保证每个到达的请求都能够被完整处理
date: 2018.02.26 21:00 +08:00
tags: 
- kubernetes
---

#### 优雅启动

很常见的一个场景，一个服务刚启动，可能会有一堆东西要加载（比如我这边需要读数据库中一堆东西）需要一些时间，而这段时间里，我不希望kubernetes 把请求打到这些还没初始化的pod上。

kubernetes提供了一个叫探针的东西，可以用来检测pod是否就绪，只有就绪的情况才会把请求打过来，如果非就绪状态，这些pod会从service的load balancer中暂时移除。

探针可以是一个command或者是一个HTTP的请求，这边使用的是一个HTTP请求的形式。需要保证程序在正常情况下可以访问/heartbeat ， 我一般会给这个接口一个success的返回。

```yaml
readinessProbe:
  httpGet:
    path: /heartbeat
    port: 8001
    scheme: HTTP
  initialDelaySeconds: 30
  timeoutSeconds: 1
  periodSeconds: 30
```

以上是一个readiness probe（就绪探针）的一般配置。

* httpGet 使用http get请求作为探针，里边配置了get请求
* initialDelaySeconds 钙素kubelet在第一次执行探针之前要等待30秒
* timeoutSeconds 探针超时时间，超过1s则认为探针失败了
* periodSeconds 探针的执行频率，默认是1s
* successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是 1。对于 liveness 必须是 1。最小值是 1。
* failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是 3。最小值是 1。



#### 优雅关闭

优雅关闭是指在pod准备关闭时，可能还需要做一些处理，比如保存数据等。这期间服务不会接受新的请求。kubernetes提供了优雅关闭的配置

```yaml
terminationGracePeriodSeconds: 60 
```

在给pod发出关闭指令时，k8s将会给应用发送SIGTERM信号，程序只需要捕获SIGTERM信号并做相应处理即可。配置为k8s会等待60秒后关闭。