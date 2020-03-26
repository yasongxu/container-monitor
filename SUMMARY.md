# 目录

## 简介
* [概述](./overview/intro.md)

## 一. 开源方案

* [第1章 采集]()
  * [Docker Stats](./opensource/agent/docker-stats.md)
  * [cAdvisor](./opensource/agent/cAdvisor.md)
  * [Heapster](./opensource/agent/Heapster.md)
   * [HPA](./opensource/agent/HPA.md)
  * [metrics-server](./opensource/agent/metrics-server.md)
  * [custom metrics](./opensource/agent/custom-metrics.md)
  * [kube-state-metrics](./opensource/agent/kube-state-metrics.md)
  * [node-exporter](./opensource/agent/node-exporter.md)
  * [Dockbix agent](./opensource/agent/dockbix.md)
  * [cortex](./opensource/agent/cortex.md)

* [第2章 Prometheus]()
   * [Prometheus](./opensource/agent/Prometheus.md)
   * [Prometheus概述](./opensource/agent/prometheus/Prometheus概述.md)
   * [Prometheus基本架构](./opensource/agent/prometheus/Prometheus基本架构.md)
   * [Prometheus部署方案](./opensource/agent/prometheus/Prometheus部署方案.md)
   * [Prometheus的配置与服务发现](./opensource/agent/prometheus/Prometheus的配置与服务发现.md)
   * [PromQL查询解析](./opensource/agent/prometheus/PromQL查询解析.md)
   * [Prometheus数据可视化](./opensource/agent/prometheus/Prometheus数据可视化.md)
   * [Prometheus存储机制](./opensource/agent/prometheus/Prometheus存储机制.md)
   * [高可用prometheus：常见问题](./prometheus/prometheus-use.md)
   * [高可用prometheus：thanos 实践](./prometheus/thanos.md)
   * [K8S常用指标分析](./prometheus/metric.md)
   * [从kubectl top看K8S监控原理](./prometheus/kubectl-top.md)

* [第3章 展示与报警]()
  * [Grafana](./opensource/dashboard/Grafana.md)
  * [Kibana](./opensource/dashboard/Kibana.md)
  * [Vizceral](./opensource/dashboard/Vizceral.md)
  * [Zabbix dashboard](./opensource/dashboard/Zabbix-dashboard.md)
  * [AlertManager](./opensource/alert/AlertManager.md)
  * [consul-alerts](./opensource/alert/consul-alerts.md)
  * [elastalert](./opensource/alert/elastalert.md)
  * [Bosun](./opensource/alert/Bosun.md)
  * [Cabot](./opensource/alert/Cabot.md)
  
## 二. 商业方案与云厂商

* [Sysdig](./enterprise/Sysdig.md)
* [DataDog](./enterprise/DataDog.md)
* [dynatrace](./enterprise/dynatrace.md)
* [Weave](./enterprise/Weave.md)
* [Cosale](./enterprise/Cosale.md)
* [freshtracks](./enterprise/freshtracks.md)
* [Sensu](./enterprise/Sensu.md)
* [netsil](./enterprise/netsil.md)
* [pingdom](./enterprise/pingdom.md)

* [Google](./cloud-provider/google.md)
* [AWS](./cloud-provider/aws.md)
* [腾讯云](./cloud-provider/tencent.md)
* [阿里云](./cloud-provider/ali.md)
* [百度云](./cloud-provider/baidu.md)
* [华为云](./cloud-provider/huawei.md)

## 四. 日志监控
 
* [ELK](./alert/ELK.md)
* [EFK](./alert/EFK.md)
* [elastalert](./alert/elastalert.md)
* [Graylog](./alert/Graylog.md)
* [docker_monitoring_logging_alerting](./alert/docker_monitoring_logging_alerting.md)

## 五. 服务监控
 
* [Jaeger](./service/Jaeger.md)
* [Zipkin](./service/Zipkin.md)
* [kubewatch](./service/kubewatch.md)
* [riemann](./service/riemann.md)


## 六. 存储后端

* [InfluxDB](./db/InfluxDB.md)
* [Kafka](./db/Kafka.md)
* [Graphite](./db/Graphite.md)
* [OpenTSDB](./db/OpenTSDB.md)
* [ElasticSearch](./db/ElasticSearch.md)

## 七. 最佳实践

* [监控方案](./opensource/prometheus/thanos.md)
* [日志方案](./solution/log.md)
* [服务监控](./solution/service.md)
* [业内方案](./solution/share/Readme.md)
  * [京东](./solution/share/jd.md)
  * [招商银行](./solution/share/zs.md)
