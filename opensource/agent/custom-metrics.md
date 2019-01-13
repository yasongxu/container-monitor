## 概述

上文metric-server提到，kubernetes的监控指标分为两种：

* Core metrics(核心指标)：从 Kubelet、cAdvisor 等获取度量数据，再由metrics-server提供给 Dashboard、HPA 控制器等使用。

* Custom Metrics(自定义指标)：由Prometheus Adapter提供API custom.metrics.k8s.io，由此可支持任意Prometheus采集到的指标。

核心指标只包含node和pod的cpu、内存等，一般来说，核心指标作HPA已经足够，但如果想根据自定义指标:如请求qps/5xx错误数来实现HPA，就需要使用自定义指标了，目前Kubernetes中自定义指标一般由Prometheus来提供，再利用k8s-prometheus-adpater聚合到apiserver，实现和核心指标（metric-server)同样的效果。

以下是官方[metrics](https://github.com/kubernetes/metrics)的项目介绍：

Resource Metrics API（核心api）

* Heapster
* Metrics Server

Custom Metrics API：

* Prometheus Adapter
* Microsoft Azure Adapter
* Google Stackdriver
* Datadog Cluster Agent

##  部署

Prometheus可以采集其它各种指标，但是prometheus采集到的metrics并不能直接给k8s用，因为两者数据格式不兼容，因此还需要另外一个组件(kube-state-metrics)，将prometheus的metrics数据格式转换成k8s API接口能识别的格式，转换以后，因为是自定义API，所以还需要用Kubernetes aggregator在主API服务器中注册，以便直接通过/apis/来访问。

文件清单：

* [node-exporter](https://github.com/prometheus/node_exporter)：prometheus的export，收集Node级别的监控数据
* [prometheus](https://prometheus.io/)：监控服务端，从node-exporter拉数据并存储为时序数据。
* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)：将prometheus中可以用PromQL查询到的指标数据转换成k8s对应的数
* [k8s-prometheus-adpater](https://github.com/directxman12/k8s-prometheus-adapter)：聚合进apiserver，即一种[custom-metrics-apiserver](https://github.com/kubernetes-incubator/custom-metrics-apiserver)实现
* 开启Kubernetes aggregator功能（参考上文metric-server）

k8s-prometheus-adapter的部署文件：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/15473590839848.jpg)

 其中创建了一个叫做cm-adapter-serving-certs的secret，包含两个值: serving.crt和serving.key，这是由apiserver信任的证书。kube-prometheus项目中的gencerts.sh和deploy.sh脚本可以创建这个secret
 
 包括secret的所有资源，都在custom-metrics命名空间下，因此需要kubectl create namespace custom-metrics
 
 以上组件均部署成功后，可以通过url获取指标
 
![](http://www.xuyasong.com/wp-content/uploads/2019/01/15473594105562.jpg)

## 基于自定义指标的HPA

使用prometheus后，pod有一些自定义指标，如http_request请求数

![](http://www.xuyasong.com/wp-content/uploads/2019/01/15473595589311.jpg)

创建一个HPA，当请求数超过每秒10次时进行自动扩容

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: http_requests
      targetAverageValue: 10
```
查看hpa

```bash
$ kubectl get hpa

NAME      REFERENCE            TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   899m / 10   2         10        2          1m
```
对pod进行施压

```bash
#install hey
$ go get -u github.com/rakyll/hey

#do 10K requests rate limited at 25 QPS
$ hey -n 10000 -q 5 -c 5 http://PODINFO_SVC_IP:9898/healthz
```

HPA发挥作用：

```bash
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  5m    horizontal-pod-autoscaler  New size: 3; reason: pods metric http_requests above target
  Normal  SuccessfulRescale  21s   horizontal-pod-autoscaler  New size: 2; reason: All metrics below target
```

## 关于k8s-prometheus-adapter

其实k8s-prometheus-adapter既包含自定义指标，又包含核心指标，即如果按照了prometheus，且指标都采集完整，k8s-prometheus-adapter可以替代metrics server。

在1.6以上的集群中，k8s-prometheus-adapter可以适配autoscaling/v2的HPA


因为一般是部署在集群内，所以k8s-prometheus-adapter默认情况下，使用[in-cluster](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#accessing-the-api-from-a-pod)的认证方式，以下是主要参数：

* lister-kubeconfig: 默认使用in-cluster方式
* metrics-relist-interval: 更新metric缓存值的间隔，最好大于等于Prometheus 的scrape interval，不然数据会为空
* prometheus-url: 对应连接的prometheus地址
* config： 一个yaml文件，配置如何从prometheus获取数据，并与k8s的资源做对应，以及如何在api接口中展示。

config文件的内容示例([参考文档](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md))

```yaml
rules:
    - seriesQuery: '{__name__=~"^container_.*",container_name!="POD",namespace!="",pod_name!=""}'
      seriesFilters: []
      resources:
        overrides:
          namespace:
            resource: namespace
          pod_name:
            resource: pod
      name:
        matches: ^container_(.*)_seconds_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>,container_name!="POD"}[1m])) by (<<.GroupBy>>)
    - seriesQuery: '{__name__=~"^container_.*",container_name!="POD",namespace!="",pod_name!=""}'
      seriesFilters:
      - isNot: ^container_.*_seconds_total$
      resources:
        overrides:
          namespace:
            resource: namespace
          pod_name:
            resource: pod
      name:
        matches: ^container_(.*)_total$
        as: ""
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>,container_name!="POD"}[1m])) by (<<.GroupBy>>)
    - seriesQuery: '{__name__=~"^container_.*",container_name!="POD",namespace!="",pod_name!=""}'
      seriesFilters:
      - isNot: ^container_.*_total$
      resources:
        overrides:
          namespace:
            resource: namespace
          pod_name:
            resource: pod
      name:
        matches: ^container_(.*)$
        as: ""
      metricsQuery: sum(<<.Series>>{<<.LabelMatchers>>,container_name!="POD"}) by (<<.GroupBy>>)
```

## 问题

为什么我看不到自定义的metric

* 检查下config配置文件，是否有选择你的metric
* 检查下采集的信息是否正确，如foo{namespace="somens",deployment="bar"}，foo这个名称的数据来自于somens的命名空间+bar这个部署
* 启动的时候加上--v=6，可以打出adapter实际的query信息


参考k8s-prometheus-adapter，可以实现自己的adapter，比如获取已有监控系统的指标，汇聚到api-server中，k8s-prometheus-adapter的实现逻辑会在后续文章中专门来讲。

本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)
