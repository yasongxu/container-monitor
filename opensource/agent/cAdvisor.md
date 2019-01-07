## 概述

  为了解决docker stats的问题(存储、展示)，谷歌开源的cadvisor诞生了，cadvisor不仅可以搜集一台机器上所有运行的容器信息，还提供基础查询界面和http接口，方便其他组件如Prometheus进行数据抓取，或者cadvisor + influxdb + grafna搭配使用。
  
   cAdvisor可以对节点机器上的资源及容器进行实时监控和性能数据采集，包括CPU使用情况、内存使用情况、网络吞吐量及文件系统使用情况
   
   Cadvisor使用Go语言开发，利用Linux的cgroups获取容器的资源使用信息，在K8S中集成在Kubelet里作为默认启动项，官方标配。
   
## 安装

* 1.使用二进制部署

```
下载二进制：https://github.com/google/cadvisor/releases/latest
本地运行：./cadvisor  -port=8080 &>>/var/log/cadvisor.log
```

* 2.使用docker部署

```
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:rw \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  google/cadvisor:latest
```
```
注意:

在Ret Hat,CentOS, Fedora 等发行版上需要传递如下参数，因为 SELinux 加强了安全策略：

--privileged=true
  
启动后访问：http://127.0.0.1:8080查看页面，/metric查看指标
```

![](http://www.xuyasong.com/wp-content/uploads/2019/01/205af1a97722816da7b542ac8f8e1b66.png)
```
* 常见指标：http://yjph83.iteye.com/blog/2394091
* 指标分析：https://luoji.live/cadvisor/cadvisor-source-code-metrics-20160927.html`
```

* 3.kubernetes中使用


```
* Daemonset部署： https://github.com/google/cadvisor/tree/master/deploy/kubernetes
* kubelet自带cadvisor监控所有节点，可以设置--cadvisor-port=8080指定端口（默认为4194）
* kubernetes 在2015-03-10 这个提交（Run cAdvisor inside the Kubelet. Victor Marmol 2015/3/10 13:39）中cAdvisor开始集成在kubelet中，目前的1.6及以后均存在
```

注意：

```
从 v1.7 开始，Kubelet metrics API 不再包含 cadvisor metrics，而是提供了一个独立的 API 接口：

* Kubelet metrics: http://127.0.0.1:8001/api/v1/proxy/nodes/<node-name>/metrics

* Cadvisor metrics: http://127.0.0.1:8001/api/v1/proxy/nodes/<node-name>/metrics/cadvisor

cadvisor 监听的端口将在 v1.12 中删除，建议所有外部工具使用 Kubelet Metrics API 替代。

```
   
## 常用搭配
 
#### 1.cAdvisor+Heapster+influxdb

![](http://www.xuyasong.com/wp-content/uploads/2019/01/bee50787d0de5749a4f11395da1aa327.png)

Heapster：在k8s集群中获取metrics和事件数据，写入InfluxDB，heapster收集的数据比cadvisor多，却全，而且存储在influxdb的也少。

```
Heapster将每个Node上的cAdvisor的数据进行汇总，然后导到InfluxDB。

Heapster的前提是使用cAdvisor采集每个node上主机和容器资源的使用情况，
再将所有node上的数据进行聚合。

这样不仅可以看到Kubernetes集群的资源情况，
还可以分别查看每个node/namespace及每个node/namespace下pod的资源情况。
可以从cluster，node，pod的各个层面提供详细的资源使用情况。
```

* InfluxDB：时序数据库，提供数据的存储，存储在指定的目录下。

* Grafana：提供了WEB控制台，自定义查询指标，从InfluxDB查询数据并展示。

 
#### **cAdvisor+Prometheus+Grafana**
访问http://localhost:8080/metrics，可以拿到cAdvisor暴露给 Prometheus的数据

![](http://www.xuyasong.com/wp-content/uploads/2019/01/d38e7f63f9a647ce217bbb9bdc77db0a.png)

其他内容参考后续的prometheus文章

## 深入解析

cAdvisor结构图

![](http://www.xuyasong.com/wp-content/uploads/2019/01/5a577e4d0a5da14b7b634b5c62264f72.png)

cadvisor地址：https://github.com/google/cadvisor

主函数逻辑：（cadvisor/cadvisor.go）

![](http://www.xuyasong.com/wp-content/uploads/2019/01/e697002b4a7a1cc8d8520e3c3f7cc1fb.png)

通过new出来的memoryStorage以及sysfs实例，创建一个manager实例，manager的interface中定义了许多用于获取容器和machine信息的函数

核心函数：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/9c27a63d31346f4e6dc592e71977d568.png)

生成manager实例的时候，还需要传递两个额外的参数，分别是

* maxHousekeepingInterval：存在内存的时间，默认60s
* allowDynamicHousekeeping：是否允许动态配置housekeeping，也就是下一次开始搜集容器信息的时间，默认true

因为需要暴露服务，所以在handler文件中，将上面生成的containerManager注册进去（cadvisor/http/handler.go)，之后就是启动manager，运行其Start方法，开始搜集信息，存储信息的循环操作。

以memory采集为例：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/197e5adaba371a4000ef9fd087dbf987.png)

具体的信息还是通过runc/libcontainer获得，libcontainer是对cgroup的封装。在/sys/fs/cgroup/memory中包含大量的了memory相关的信息（参考docker原生监控文章）

![](http://www.xuyasong.com/wp-content/uploads/2019/01/8d1b0d821546002af20648d131858457.png)

Prometheus的收集器（cadvisor/metrics/prometheus.go）

![](http://www.xuyasong.com/wp-content/uploads/2019/01/10f40e80b299ff5df8a99acca63c2644.png)

更多源码参考文章：https://luoji.live/categories/cadvisor/

## 总结

优缺点：

* 优点：谷歌开源产品，监控指标齐全，部署方便，而且有官方的docker镜像。
* 缺点：是集成度不高，默认只在本地保存1分钟数据，但可以集成InfluxDB等存储

备注：

```
爱奇艺参照cadvisor开发的dadvisor，数据写入graphite，
等同于cadvisor+influxdb，但dadvisor并没有开源
```

