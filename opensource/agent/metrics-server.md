# 概述

从 v1.8 开始，资源使用情况的监控可以通过 Metrics API的形式获取，具体的组件为Metrics Server，用来替换之前的heapster，heapster从1.11开始逐渐被废弃。

Metrics-Server是集群核心监控数据的聚合器，从 Kubernetes1.8 开始，它作为一个 Deployment对象默认部署在由kube-up.sh脚本创建的集群中，如果是其他部署方式需要单独安装，或者咨询对应的云厂商。

## Metrics API

介绍Metrics-Server之前，必须要提一下Metrics API的概念

Metrics API相比于之前的监控采集方式(hepaster)是一种新的思路，官方希望核心指标的监控应该是稳定的，版本可控的，且可以直接被用户访问(例如通过使用 kubectl top 命令)，或由集群中的控制器使用(如HPA)，和其他的Kubernetes APIs一样。

官方废弃heapster项目，就是为了将核心资源监控作为一等公民对待，即像pod、service那样直接通过api-server或者client直接访问，不再是安装一个hepater来汇聚且由heapster单独管理。

假设每个pod和node我们收集10个指标，从k8s的1.6开始，支持5000节点，每个节点30个pod，假设采集粒度为1分钟一次，则：

```
10 x 5000 x 30 / 60 = 25000 平均每分钟2万多个采集指标
```

因为k8s的api-server将所有的数据持久化到了etcd中，显然k8s本身不能处理这种频率的采集，而且这种监控数据变化快且都是临时数据，因此需要有一个组件单独处理他们，k8s版本只存放部分在内存中，于是metric-server的概念诞生了。

其实hepaster已经有暴露了api，但是用户和Kubernetes的其他组件必须通过master proxy的方式才能访问到，且heapster的接口不像api-server一样，有完整的鉴权以及client集成。这个api现在还在alpha阶段（18年8月），希望能到GA阶段。类api-server风格的写法：[generic apiserver](https://github.com/kubernetes/apiserver)

有了Metrics Server组件，也采集到了该有的数据，也暴露了api，但因为api要统一，如何将请求到api-server的`/apis/metrics`请求转发给Metrics Server呢，解决方案就是：[kube-aggregator](https://github.com/kubernetes/kube-aggregator),在k8s的1.7中已经完成，之前Metrics Server一直没有面世，就是耽误在了kube-aggregator这一步。

kube-aggregator（聚合api）主要提供：

* Provide an API for registering API servers.
* Summarize discovery information from all the servers.
* Proxy client requests to individual servers.

详细设计文档：[参考链接](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/aggregated-api-servers.md)

metric api的使用：

* Metrics API 只可以查询当前的度量数据，并不保存历史数据

* Metrics API URI 为 /apis/metrics.k8s.io/，在 k8s.io/metrics 维护

* 必须部署 metrics-server 才能使用该 API，metrics-server 通过调用 Kubelet Summary API 获取数据

如： 

```
http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes

http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes/<node-name>

http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespace/<namespace-name>/pods/<pod-name>
```

## Metrics-Server

Metrics server定时从Kubelet的Summary API(类似/ap1/v1/nodes/nodename/stats/summary)采集指标信息，这些聚合过的数据将存储在内存中，且以metric-api的形式暴露出去。

Metrics server复用了api-server的库来实现自己的功能，比如鉴权、版本等，为了实现将数据存放在内存中吗，去掉了默认的etcd存储，引入了内存存储（即实现[Storage interface](https://github.com/kubernetes/apiserver/blob/master/pkg/registry/rest/rest.go))。因为存放在内存中，因此监控数据是没有持久化的，可以通过第三方存储来拓展，这个和heapster是一致的。

![](http://www.xuyasong.com/wp-content/uploads/2019/01/03659112b8502f2a2ab7b7df8b735782.png)

Metrics server出现后，新的​Kubernetes 监控架构将变成上图的样子

* 核心流程（黑色部分）：这是 Kubernetes正常工作所需要的核心度量，从 Kubelet、cAdvisor 等获取度量数据，再由metrics-server提供给 Dashboard、HPA 控制器等使用。

* 监控流程（蓝色部分）：基于核心度量构建的监控流程，比如 Prometheus 可以从 metrics-server 获取核心度量，从其他数据源（如 Node Exporter 等）获取非核心度量，再基于它们构建监控告警系统。

官方地址：https://github.com/kubernetes-incubator/metrics-server

## 使用

如上文提到的，metric-server是扩展的apiserver，依赖于kube-aggregator，因此需要在apiserver中开启相关参数。

```
--requestheader-client-ca-file=/etc/kubernetes/certs/proxy-ca.crt
--proxy-client-cert-file=/etc/kubernetes/certs/proxy.crt
--proxy-client-key-file=/etc/kubernetes/certs/proxy.key
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User

```

安装文件下载地址：[1.8+](https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B)，注意更换镜像地址为国内镜像

`kubectl create -f metric-server/`
![](http://www.xuyasong.com/wp-content/uploads/2019/01/15473067971765.jpg)

安装成功后，访问地址api地址为：
![](http://www.xuyasong.com/wp-content/uploads/2019/01/15473069387068.jpg)

   Metrics Server的资源占用量会随着集群中的Pod数量的不断增长而不断上升，因此需要
addon-resizer垂直扩缩这个容器。addon-resizer依据集群中节点的数量线性地扩展Metrics Server，以保证其能够有能力提供完整的metrics API服务。具体参考：[链接](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer)

### 其他

基于Metrics Server的HPA：[参考链接](http://blog.51cto.com/ylw6006/2115087)

kubernetes的新监控体系中，metrics-server属于Core metrics(核心指标)，提供API metrics.k8s.io，仅提供Node和Pod的CPU和内存使用情况。而其他Custom Metrics(自定义指标)由Prometheus等组件来完成，后续文章将对自定义指标进行解析。

本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)
