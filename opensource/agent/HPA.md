## 概述

Horizontal Pod Autoscaling，简称HPA，是Kubernetes中实现POD水平自动伸缩的功能。自动扩展主要分为两种:

* 水平扩展(scale out)，针对于实例数目的增减
* 垂直扩展(scal up)，即单个实例可以使用的资源的增减, 比如增加cpu和增大内存 

HPA属于前者。它可以根据CPU使用率或应用自定义metrics自动扩展Pod数量(支持 replication controller、deployment 和 replica set)

## 监控数据获取

* Heapster: heapster收集Node节点上的cAdvisor数据，并按照kubernetes的资源类型来集合资源。但是在v1.11中已经被废弃（heapster监控数据可用，但HPA不再从heapster拿数据）

* metric-server: 在v1.8版本中引入，官方将其作为heapster的替代者。metric-server依赖于kube-aggregator，因此需要在apiserver中开启相关参数。v1.11中HPA从metric-server获取监控数据

## 工作流程

* 1.创建HPA资源，设定目标CPU使用率限额，以及最大、最小实例数, 一定要设置Pod的资源限制参数: request, 负责HPA不会工作。
* 2.控制管理器每隔30s(可以通过–horizontal-pod-autoscaler-sync-period修改)查询metrics的资源使用情况
* 3.然后与创建时设定的值和指标做对比(平均值之和/限额)，求出目标调整的实例个数
* 4.目标调整的实例数不能超过1中设定的最大、最小实例数，如果没有超过，则扩容；超过，则扩容至最大的实例个数

## 如何部署：

* 1.6-1.10版本默认使用heapster
* 1.11版本及以上默认使用metric-server

1.部署和运行php-apache并将其暴露成为服务

![](http://www.xuyasong.com/wp-content/uploads/2019/01/4115bcf5b70e874b486c96a8743fa45a.png)

2.创建HPA

![](http://www.xuyasong.com/wp-content/uploads/2019/01/69ebc98a51677feeb8276ff524635a38.png)

如果为1.8及以下的k8s集群，指标正常，如果为1.11集群，需要执行如下操作。

![](http://www.xuyasong.com/wp-content/uploads/2019/01/f34cb8184245e0ad4ece9b0ed1fe36bc.png)

4.向php-apache服务增加负载，验证自动扩缩容
 启动一个容器，并通过一个循环向php-apache服务器发送无限的查询请求（请在另一个终端中运行以下命令）
 
![](http://www.xuyasong.com/wp-content/uploads/2019/01/433ed01fba6be0167ba9fe4155abd9db.png)

5.观察HPA是否生效

![](http://www.xuyasong.com/wp-content/uploads/2019/01/520c5b9289764dc539ee6fd9a7b23e85.png)
