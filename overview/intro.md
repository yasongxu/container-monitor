# 概述

  随着越来越多的线上服务docker化，对容器的监控、报警变得越来越重要，容器监控有多种形态，有些是开源的（如promethues)，而另一些则是商业性质的(如Weave)，有些是集成在云厂商一键部署的（Rancher、谷歌云），有些是手动配置的，可谓百花齐放。
  
  本文将对现有的容器监控方案进行总结对比，监控解决方案的数量之多令人望而生畏，新的解决方案又不断涌现，下面将从开源方案（采集、展示、告警）、商业方案、云厂商、主机监控、日志监控、服务监控等方面进行列举，此篇为概述篇，包含汇总列表及脑图，具体分析将在后面补充更新，欢迎指正、补充。

# 方案汇总
  
#### 一. 开源方案

1.采集：

* Docker Stats
* cAdvisor
* Heapster
* metrics-server
* Custom Metrics
* kube-state-metrics
* node-exporter
* Prometheus
* Dockbix agent
* cortex

2.展示：

* Grafana
* Kibana
* Vizceral
* mozaik
* Zabbix dashboard

3.报警：

* AlertManager
* consul-alerts
* elastalert
* Bosun
* Cabot

#### 二. 商业方案

* Sysdig
* DataDog
* dynatrace
* Weave
* Thanos
* Cosale
* freshtracks
* newrelic
* Sensu
* netsil
* pingdom
    
#### 三. 云厂商

* Google cloud
* AWS
* 腾讯云
* 阿里云
* 百度云
* 华为云

#### 四. 主机监控

* Zabbix
* nagios
* netdata

#### 五. 日志监控

* ELK Stack
* EFK Stack
* elastalert
* Graylog
* docker_monitoring_logging_alerting

#### 六. 服务监控

* 	Jaeger
* 	Zipkin
* 	kubewatch
* 	riemann
    
##### 七. 存储后端

* 	InfluxDB
* 	Kafka
* 	Graphite
* 	OpenTSDB
* 	ElasticSearch
  
# 脑图
![](http://www.xuyasong.com/wp-content/uploads/2019/01/26dc8b2a26b9a9a8604f817ed9876054-1.png)
