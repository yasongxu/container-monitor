### 一.概述

cortex：一个支持多租户、水平扩展的prometheus服务。

当时调研cortex其实是因为看到了Weave Cloud这个商业产品中的监控模块介绍，weave也叫weave works，官方地址是：[https://cloud.weave.works](https://cloud.weave.works)，是一个专注于容器微服务的paas平台。

WeaveCloud在监控模块最大化利用了Prometheus，并在其基础上添加了很多组件，实现了多租户管理、高可用的监控集群。其使用的核心监控组件就是**cortex**。

本文主要分享的是cortex的运行机制，关于Weave Cloud的产品定位和功能可以看下后续的文章:[商业方案-weave work]()


Cortex是一个CNCF的沙盒项目，目前被几个线上产品使用：Weave Cloud、GrafanaCloud和FreshTracks.io

**为什么不直接运行Prometheus，而用Cortex？**

ps:来自cortex kubecon大会演讲

* 作为服务，cortex提供了鉴权和访问控制
* 数据永久保留，状态能够被管理
* 提供持久化、高可用、伸缩性
* 提供更好的查询效率，尤其是长查询


### 二.主要功能

针对以上需求，Cortex提供的主要功能或特色如下：

* 支持多租户：Prometheus本身没有的租户概念。这意味着，它无法对特定于租户的数据访问和资源使用配额，提供任何形式的细粒度控制。Cortex可以从多个独立的prometheus实例中获取数据，并按照租户管理。
* 长期存储：基于远程写入机制，支持四种开箱即用的长期存储系统：AWS DynamoDB、AWS S3、Apache Cassandra和Google Cloud Bigtable。
* 全局视图：提供所有prometheus server 整合后的时间序列数据的单一，一致的“全局”视图。
* 高可用：提供服务实例的水平扩展、联邦集群等
* 最大化利用了Prometheus

相似的竞品：

* Prometheus + InfluxDB：使用InfluxData
* Prometheus + Thanos：长期存储、全局视图
* Timbala：多副本、全局视图，作者是Matt Bostock
* M3DB：自动扩缩容，来自uber

##### 产品形态

ps:来自weave work上试用监控模块时的截图

* 1.安装监控的agent:

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543000032028.jpg)

* 2.概览视图

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543000291498.jpg)

* 3.资源监控面板

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543000496622.jpg)


* 4.监控详情页面

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543000699174.jpg)


* 5.添加监控

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543000838449.jpg)

* 6.配置报警

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543001007109.jpg)

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543001083178.jpg)

在k8s集群中部署所需要的yaml列表为：

[https://github.com/weaveworks/cortex/tree/master/k8s](https://github.com/weaveworks/cortex/tree/master/k8s
)

部署的agent时的脚本内容是：

```shell
#!/bin/sh
set -e
# Create a temporary file for the bootstrap binary
TMPFILE="$(mktemp -qt weave_bootstrap.XXXXXXXXXX)" || exit 1
finish(){
  # Send only when this script errors out
  # Filter out the bootstrap errors
  if [ $? -ne 111 ] && [ $? -ne 0 ]; then
    curl -s >/dev/null 2>/dev/null -H "Accept: application/json" -H "Authorization: Bearer $token" -X POST -d \
        '{"type": "onboarding_failed", "messages": {"browser": { "type": "onboarding_failed", "text": "Installation of Weave Cloud agents did not finish."}}}' \
        https://cloud.weave.works/api/notification/external/events || true
  fi
  # Arrange for the bootstrap binary to be deleted
  rm -f "$TMPFILE"
}
# Call finish function on exit
trap finish EXIT
# Parse command-line arguments
for arg in "$@"; do
    case $arg in
        --token=*)
            token=$(echo $arg | cut -d '=' -f 2)
            ;;
    esac
done
if [ -z "$token" ]; then
    echo "error: please specify the instance token with --token=<TOKEN>"
    exit 1
fi
# Notify installation has started
curl -s >/dev/null 2>/dev/null -H "Accept: application/json" -H "Authorization: Bearer $token" -X POST -d \
    '{"type": "onboarding_started", "messages": {"browser": { "type": "onboarding_started", "text": "Installation of Weave Cloud agents has started"}}}' \
    https://cloud.weave.works/api/notification/external/events || true
# Get distribution
unamestr=$(uname)
if [ "$unamestr" = 'Darwin' ]; then
    dist='darwin'
elif [ "$unamestr" = 'Linux' ]; then
    dist='linux'
else
  echo "This OS is not supported"
  exit 1
fi
# Download the bootstrap binary
echo "Downloading the Weave Cloud installer...  "
curl -Ls "https://get.weave.works/bootstrap?dist=$dist" >> "$TMPFILE"
# Make the bootstrap binary executable
chmod +x "$TMPFILE"
# Execute the bootstrap binary
"$TMPFILE" "--scheme=https" "--wc.launcher=get.weave.works" "--wc.hostname=cloud.weave.works" "--report-errors" "$@"
```

### 三.实现原理

Cortex与Prometheus的交互图：

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543003052174.jpg)

原理图：

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543010104189.jpg)


Cortex中各组件的作用：

* Retrieval：采集组件，运行在用户k8s集群上，从用户应用中拉取监控指标，并把这些数据推送给云平台的服务

* Frontend:  负载均衡/路由转发/权限认证，接受Retrieval发送来的请求，这里用的nginx

* Distributor：分发器，把用户推送来的监控指标，按照用户id、指标名称、标签做一致性hash，然后并行交给后面的多个ingester处理(grpc交互)。是监控数据写入的第一站

* Ingester：处理器，将监控数据保存到promtheus中，高度定制了MemorySeriesStorage模块，分块存储、写入内存并索引（使用AWS的DynamoDB产品），最后写入磁盘

* 读写分离：ingest和query分开为两个服务

Cortex由多个可水平扩展的微服务组成。每个微服务使用最合适的技术进行水平缩放; 大多数是无状态的，而有些（即Retrieval）是半有状态的并且依赖于一致性哈希

Prometheus实例从各种目标中抓取样本，然后将它们推送到Cortex（使用Prometheus的远程写入API),并对发送的Protocol Buffers序列化数据进行Snappy压缩。

Cortex要求每个HTTP请求都带有一个header，用于指定请求的租户ID。请求身份验证和授权由外部反向代理处理。

传入的样本（来自Prometheus的写入）由Distributor处理，而传入的读取（PromQL查询）由查询前端处理。


查询缓存：

查询时会缓存存查询结果，并在后续查询中复用它们。如果缓存的结果不完整，则查询前端计算所需的子查询并在下游查询器上并行执行它们。

并发查询：

查询作业接受来自查询器的gRPC流请求，为了实现高可用性，建议您运行多个前端，且前端数量少于查询器数量。在大多数情况下，两个应该足够了。

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15543008126966.jpg)


* 技术方案实现讨论细节：[https://goo.gl/prdUYV](https://goo.gl/prdUYW)
* CNCF TOC：[https://docs.google.com/presentation/d/190oIFgujktVYxWZLhLYN4q8p9dtQYoe4sxHgn4deBSI/edit#slide=id.g3b8e2d6f7e_0_101](https://docs.google.com/presentation/d/190oIFgujktVYxWZLhLYN4q8p9dtQYoe4sxHgn4deBSI/edit#slide=id.g3b8e2d6f7e_0_101)

* kubecon 大会讲稿：[https://kccna18.sched.com/event/GrXL/cortex-infinitely-scalable-prometheus-bryan-boreham-weaveworks](https://kccna18.sched.com/event/GrXL/cortex-infinitely-scalable-prometheus-bryan-boreham-weaveworks)

本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)
