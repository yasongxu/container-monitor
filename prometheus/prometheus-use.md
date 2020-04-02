监控系统的历史悠久，是一个很成熟的方向，而Prometheus作为新生代的开源监控系统，慢慢成为了云原生体系的事实标准，也证明了其设计很受欢迎。本文主要分享在prometheus实践中遇到的一些问题和思考

## 几点原则

* 监控是基础设施，目的是为了解决问题，不要只朝着大而全去做，尤其是不必要的指标采集，浪费人力和存储资源（To B商业产品例外）
* 需要处理的告警才发出来，发出来的告警必须得到处理
* 简单的架构就是最好的架构，业务系统都挂了，监控也不能挂，Google SRE里面也说避免使用magic系统，例如机器学习报警阈值、自动修复之类。这一点见仁见智吧，感觉很多公司都在搞智能AI运维

## prometheus 的局限

* prometheus是基于metric的监控，不适用于日志（logs）、事件(event)、调用链(tracing)
* prometheus默认是pull模型，合理规划你的网络，尽量不用pushgateway转发
* 对于集群化、水平扩展，官方和社区都没有银弹，合理选择 federate、cortex、thanos
* 监控系统一般 可用性>一致性，这个后面说 thanos 的时候会提到

## 合理选择黄金指标

我们应该关注哪些指标？Google在“SRE Handbook”中提出了“四个黄金信号”：延迟、流量、错误数、饱和度。实际操作中可以使用USE或RED 方法作为指导，USE用于资源，RED用于服务

* USE 方法：Utilization、Saturation、Errors
* RED 方法：Rate、Errors、Duration

对USE和RED的阐述可以参考[容器监控实践—K8S常用指标分析](http://www.xuyasong.com/?p=1717)这篇文章

## 采集组件all in one 

prometheus 体系中 exporter 都是独立的，每个组件各司其职，如机器资源用 node-exporter，gpu 有NVIDIA exporter等等，但是 exporter 越多，运维压力越大，尤其是对 agent做资源控制、版本升级。我们尝试对一些exporter进行组合，方案有二：

1. 通过主进程拉起n个exporter 进程，仍然可以跟着社区版本更新
2. 用telegraf来支持各种类型的 input，n 合 1

另外，node-exporter 不支持进程监控，可以加一个process-exporter，也可以用上边提到的telegraf。


## k8s 1.16中cadvisor的指标兼容问题

在 k8s 1.16版本，cadvisor的指标去掉了pod_name 和 container_name的label，替换为了pod 和 container。如果你之前用这两个 label 做查询或者 grafana 绘图，得更改下 sql 了。因为我们一直支持多个 k8s 版本，就通过 relabel配置继续保留了原来的**_name

```yaml
metric_relabel_configs:
- source_labels: [container]
  regex: (.+)
  target_label: container_name
  replacement: $1
  action: replace
- source_labels: [pod]
  regex: (.+)
  target_label: pod_name
  replacement: $1
  action: replace
```

注意要用metric_relabel_configs，不是relabel_configs，采集后做的replace。

## prometheus集群内与集群外部署

prometheus 如果部署在k8s集群内采集是很方便的，用官方给的yaml就可以，但我们因为权限和网络需要部署在集群外，二进制运行，专门划了几台高配服务器运行监控组件。

以 pod 方式运行在集群内是不需要证书的（in-cluster 模式），但集群外需要声明 token之类的证书，并替换__address__。例如：

```yaml
kubernetes_sd_configs:
- api_server: https://xx:6443
  role: node
  bearer_token_file: token/xx.token
  tls_config:
    insecure_skip_verify: true
relabel_configs:
- separator: ;
  regex: __meta_kubernetes_node_label_(.+)
  replacement: $1
  action: labelmap
- separator: ;
  regex: (.*)
  target_label: __address__
  replacement: xx:6443
  action: replace
- source_labels: [__meta_kubernetes_node_name]
  separator: ;
  regex: (.+)
  target_label: __metrics_path__
  replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
  action: replace
```

上面是通过默认配置中通过apiserver proxy 到 let，如果网络能通，其实也可以直接把kubelet的10255作为 target，规模大的时候还减轻了 apiserver 的压力，不过这种方式就要写服务发现来更新 node列表了。

## gpu 指标的获取

nvidia-smi可以查看机器上的gpu资源，而cadvisor 其实暴露了metric来表示 容器使用 gpu 情况，

```bash
container_accelerator_duty_cycle
container_accelerator_memory_total_bytes
container_accelerator_memory_used_bytes
```

如果要更详细的gpu 数据，可以安装[dcgm exporter](https://github.com/NVIDIA/gpu-monitoring-tools/tree/master/exporters/prometheus-dcgm)，不过k8s 1.13 才能支持


## 更改 prometheus的显示时区

prometheus为避免时区混乱，在所有组件中专门使用Unix time和UTC进行显示。不支持在配置文件中设置时区，也不能读取本机/etc/timezone时区。

其实这个限制是不影响使用的：

* 如果做可视化，grafana是可以做时区转换的
* 如果是调接口，拿到了数据中的时间戳，你想怎么处理都可以
* 如果因为prometheus 自带的 ui不是本地时间，看着不舒服， [2.16 版本](https://github.com/prometheus/prometheus/commit/d996ba20ec9c7f1808823a047ed9d5ce96be3d8f)的新版 webui已经引入了local timezone 的选项。区别见下图
* 如果你仍然想改prometheus 代码来适应自己的时区，可以参考[这篇文章](https://zhangguanzhang.github.io/2019/09/05/prometheus-change-timezone/)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/2e415b33-c061-4dec-8433-777ba4edcb8c.jpg?x-oss-process=style/watermark)


![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/b1afc5c8-b49f-44f8-bc06-086361c83024.jpg?x-oss-process=style/watermark)


关于 timezone 的讨论，可以看这个[issue](https://github.com/prometheus/prometheus/issues/500)

## 如何采集 lb 后面的rs 的 metric

假如你有一个负载均衡lb，但网络上prometheus 只能访问到 lb本身，访问不到后面的rs，应该如何采集 rs 暴露的 metric？

* rs 的服务加 sidecar proxy，或者本机增加 proxy 组件，保证prometheus能访问到
* lb 增加/ backend1和/ backend2请求转发到两个单独的后端，再由prometheus访问 lb 采集

## 版本

prometheus当前最新版本为 2.16，prometheus还在不断迭代，因此尽量用最新版，1.x版本就不用考虑了。

2.16 版本上有一套实验 UI，可以查看TSDB的状态，包括top 10的label、metric

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/9668d944-b7b7-425e-b86b-ae24c5aa2f0c.jpg?x-oss-process=style/watermark)

## prometheus 大内存问题

随着规模变大，prometheus需要的cpu和内存都会升高，内存一般先达到瓶颈，这个时候要么加内存，要么集群分片减少单机指标。这里我们先讨论单机版prometheus的内存问题

原因：

* prometheus 的内存消耗主要是因为每隔2小时做一个 block 数据落盘，落盘之前所有数据都在内存里面，因此和采集量有关。
* 加载历史数据时，是从磁盘到内存的，查询范围越大，内存越大。这里面有一定的优化空间
* 一些不合理的查询条件也会加大内存，如 group、大范围rate

我的指标需要多少内存：

* 作者给了一个计算器，设置指标量、采集间隔之类的，计算 prometheus 需要的理论内存值：https://www.robustperception.io/how-much-ram-does-prometheus-2-x-need-for-cardinality-and-ingestion

以我们的一个 promserver为例，本地只保留 2 小时数据，95 万 series，大概占用的内存如下：
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/596f6d60-2e37-460a-8da5-9dfd07b7170d.jpg?x-oss-process=style/watermark)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/5a5ea166-f9df-49d1-ab79-02a925c11322.jpg?x-oss-process=style/watermark)

有什么优化方案：

* sample 数量超过了 200 万，就不要单实例了，做下分片，然后通过victoriametrics，thanos，trickster等方案合并数据
* 评估哪些metric 和 label占用较多，去掉没用的指标。2.14 以上可以看 [tsdb 状态](https://www.google.com/url?q=https%3A%2F%2Fprometheus.io%2Fdocs%2Fprometheus%2Flatest%2Fquerying%2Fapi%2F%23tsdb-stats&sa=D&sntz=1&usg=AFQjCNFE5AzQxyhzt8SqQLHPUySZl3lNNw)
* 查询时尽量避免大范围查询，注意时间范围和 step 的比例，慎用 group
* 如果需要关联查询，先想想能不能通过 relabel 的方式给原始数据多加个 label，一条sql 能查出来的何必用join，时序数据库不是关系数据库。

prometheus 内存占用分析：

* 通过 pprof分析：https://www.robustperception.io/optimising-prometheus-2-6-0-memory-usage-with-pprof
* 1.x 版本的内存：https://www.robustperception.io/how-much-ram-does-my-prometheus-need-for-ingestion

相关 issue：

* https://groups.google.com/forum/#!searchin/prometheus-users/memory%7Csort:date/prometheus-users/q4oiVGU6Bxo/uifpXVw3CwAJ
* https://github.com/prometheus/prometheus/issues/5723
* https://github.com/prometheus/prometheus/issues/1881

## prometheus 容量规划

容量规划除了上边说的内存，还有磁盘存储规划，这和你的 prometheus 的架构方案有关

* 如果是单机prometheus，计算本地磁盘使用量
* 如果是 remote-write，和已有的 tsdb 共用即可。
* 如果是 thanos 方案，本地磁盘可以忽略（2h)，计算对象存储的大小就行。

Prometheus每2小时将已缓冲在内存中的数据压缩到磁盘上的块中。包括chunks, indexes, tombstones 和metadata，这些占用了一部分存储空间。一般情况下，Prometheus中存储的每一个样本大概占用1-2字节大小（1.7byte）。可以通过promql来查看每个样本平均占用多少空间：

 ```bash
 rate(prometheus_tsdb_compaction_chunk_size_bytes_sum[1h])
/ 
  rate(prometheus_tsdb_compaction_chunk_samples_sum[1h])
  
{instance="0.0.0.0:8890", job="prometheus"}  1.252747585939941
 ```

如果大致估算本地磁盘大小，可以通过以下公式：

```bash
磁盘大小 = 保留时间 * 每秒获取样本数 * 样本大小
```

保留时间(retention_time_seconds)和样本大小(bytes_per_sample)不变的情况下，如果想减少本地磁盘的容量需求，只能通过减少每秒获取样本数(ingested_samples_per_second)的方式。

查看当前每秒获取的样本数：
```bash
rate(prometheus_tsdb_head_samples_appended_total[1h])
```

有两种手段，一是减少时间序列的数量，二是增加采集样本的时间间隔。考虑到Prometheus会对时间序列进行压缩，因此减少时间序列的数量效果更明显.

举例说明：

* 采集频率 30s，机器数量1000，metric种类6000，1000*6000*2*60*24 约 200 亿，30G左右磁盘
* 只采集需要的指标，如 match[], 或者统计下最常使用的指标，性能最差的指标

以上磁盘容量并没有把 WAL 文件算进去，WAL 文件(raw data)Prometheus 官方文档中说明至少会保存3个 write-ahead log files，每一个最大为128M(实际运行发现数量会更多)

因为我们使用了 thanos 的方案，所以本地磁盘只保留2h 热数据。WAL 每2小时生成一份block文件，block文件每2小时上传对象存储，本地磁盘基本没有压力。

关于prometheus存储机制，可以看[这篇](http://www.xuyasong.com/?p=1601)

## 对 apiserver的 性能影响

如果你的 prometheus 使用了kubernetes_sd_config做服务发现，请求一般会经过集群的 apiserver，随着规模的变大，需要评估下对 apiserver性能的影响，尤其是proxy失败的时候，会导致cpu 升高。当然了，如果单k8s集群规模太大，一般都是拆分集群，不过随时监测下 apiserver 的进程变化还是有必要的。

在监控cadvisor、docker、kube-proxy 的 metric 时，我们一开始选择从 apiserver proxy 到节点的对应端口，统一设置比较方便，但后来还是改为了直接拉取节点，apiserver 仅做服务发现。


## rate 的计算逻辑

prometheus 中的counter类型主要是为了 rate 而存在的，即计算速率，单纯的counter计数意义不大，因为counter一旦重置，总计数就没有意义了。

rate会自动处理counter重置的问题，counter一般都是一直变大的，例如一个exporter启动，然后崩溃了。本来以每秒大约10的速率递增，但仅运行了半个小时，则速率（x_total [1h]）将返回大约每秒5的结果。另外，counter的任何减少也会被视为counter重置。例如，如果时间序列的值为[5,10,4,6]，则将其视为[5,10,14,16]。

rate值很少是精确的。由于针对不同目标的抓取发生在不同的时间，因此随着时间的流逝会发生抖动，query_range计算时很少会与抓取时间完美匹配，并且抓取有可能失败。面对这样的挑战，rate的设计必须是健壮的。

rate并非想要捕获每个增量，因为有时候增量会丢失，例如实例在抓取间隔中挂掉。如果counter的变化速度很慢，例如每小时仅增加几次，则可能会导致【假象】。比如出现一个counter时间序列，值为100，rate就不知道这些增量是现在的值，还是目标已经运行了好几年并且才刚刚开始返回。

建议将rate计算的范围向量的时间至少设为抓取间隔的四倍。这将确保即使抓取速度缓慢，且发生了一次抓取故障，您也始终可以使用两个样本。此类问题在实践中经常出现，因此保持这种弹性非常重要。例如，对于1分钟的抓取间隔，您可以使用4分钟的rate 计算，但是通常将其四舍五入为5分钟。

如果 rate 的时间区间内有数据缺失，他会基于趋势进行推测，比如：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/eedead36-3aa5-4a1d-831d-dc53a80630b1.jpg?x-oss-process=style/watermark)

详细的内容可以看下这个[视频](https://www.youtube.com/watch?reload=9&v=67Ulrq6DxwA)

## 反直觉的 p95统计

histogram_quantile 是 Prometheus 常用的一个函数，比如经常把某个服务的 P95 响应时间来衡量服务质量。不过它到底是什么意思很难解释得清，特别是面向非技术的同学，会遇到很多“灵魂拷问”。

我们常说 P95（p99,p90都可以） 响应延迟是 100ms，实际上是指对于收集到的所有响应延迟，有 5% 的请求大于 100ms，95% 的请求小于 100ms。Prometheus 里面的 histogram_quantile 函数接收的是 0-1 之间的小数，将这个小数乘以 100 就能很容易得到对应的百分位数，比如 0.95 就对应着 P95，而且还可以高于百分位数的精度，比如 0.9999。

当你用histogram_quantile画出响应时间的趋势图时，可能会被问：为什么p95大于或小于我的平均值？

正如中位数可能比平均数大也可能比平均数小，P99 比平均值小也是完全有可能的。通常情况下 P99 几乎总是比平均值要大的，但是如果数据分布比较极端，最大的 1% 可能大得离谱从而拉高了平均值。一种可能的例子：

```bash
1, 1, ... 1, 901 // 共 100 条数据，平均值=10，P99=1
```

服务 X 由顺序的 A，B 两个步骤完成，其中 X 的 P99 耗时 100ms，A 过程 P99 耗时 50ms，那么推测 B 过程的 P99 耗时情况是？

直觉上来看，因为有 X=A+B，所以答案可能是 50ms，或者至少应该要小于 50ms。实际上 B 是可以大于 50ms 的，只要 A 和 B 最大的 1% 不恰好遇到，B 完全可以有很大的 P99：

```bash
A = 1, 1, ... 1,  1,  1,  50,  50 // 共 100 条数据，P99=50
B = 1, 1, ... 1,  1,  1,  99,  99 // 共 100 条数据，P99=99
X = 2, 2, ... 1, 51, 51, 100, 100 // 共 100 条数据，P99=100

```

```bash
如果让 A 过程最大的 1% 接近 100ms，我们也能构造出 P99 很小的 B：
A = 50, 50, ... 50,  50,  99 // 共 100 条数据，P99=50
B =  1,  1, ...  1,   1,  50 // 共 100 条数据，P99=1
X = 51, 51, ... 51, 100, 100 // 共 100 条数据，P99=100
```

所以我们从题目唯一能确定的只有 B 的 P99 应该不能超过 100ms，A 的 P99 耗时 50ms 这个条件其实没啥用。


类似的疑问很多，因此对于histogram_quantile函数，可能会产生反直觉的一些结果，最好的处理办法是不断试验调整你的 bucket 的值，保证更多的请求时间落在更细致的区间内，这样的请求时间才有统计意义。

## 慢查询问题

* promql 的基础知识看这篇[文章](http://www.xuyasong.com/?p=1578)

prometheus 提供了自定义的promql作为查询语句，在 graph上调试的时候，会告诉你这条 sql 的返回时间，如果太慢你就要注意了，可能是你的用法出现了问题。

评估 prometheus 的整体响应时间，可以用这个默认指标:

```bash
prometheus_engine_query_duration_seconds{}
```

一般情况下响应过慢都是promql 使用不当导致，或者指标规划有问题，如：

* 大量使用 join 来组合指标或者增加 label，如将 kube-state-metric 中的一些 meta label和 node-exporter 中的节点属性 label加入到 cadvisor容器 数据里，像统计 pod 内存使用率并按照所属节点的机器类型分类，或按照所属rs 归类。
* 范围查询时，大的时间范围，step 值却很小，导致查询到的数量量过大。
* rate会自动处理counter重置的问题，最好由 promql 完成，不要自己拿出来全部元数据在程序中自己做rate计算。
* 在使用 rate 时，range duration要大于等于[step](https://www.robustperception.io/step-and-query_range)，否则会丢失[部分数据](https://chanjarster.github.io/post/p8s-step-param/)
* prometheus 是有基本预测功能的，如`deriv`和`predict_linear`(更准确)可以根据已有数据预测未来趋势
* 如果比较复杂且耗时的sql，可以使用record rule减少指标数量，并使查询效率更高，但不要什么指标都加record，一半以上的 metric 其实不太会查询到。同时 label 中的值不要加到record rule 的 name 中。


## 高基数问题Cardinality

高基数是数据库避不开的一个话题，对于mysql这种db来讲，基数是指特定列或字段中包含的唯一值的数量。基数越低，列中重复的元素越多。对于时序数据库而言，就是tags、label 这种标签值的数量多少。

比如 prometheus 中如果有一个指标 `http_request_count{method="get",path="/abc",originIP="1.1.1.1"}`表示访问量，method 表示请求方法，originIP是客户端 IP，method的枚举值是有限的，但originIP却是无限的，加上其他 label 的排列组合就无穷大了，也没有任何关联特征，因此这种高基数不适合作为metric 的 label，真要的提取originIP，应该用日志的方式，而不是 metric 监控

时序数据库会为这些 label建立索引，以提高查询性能，以便您可以快速找到与所有指定标签匹配的值。如果值的数量过多，索引是没有意义的，尤其是做p95 等计算的时候，要扫描大量 series 数据

官方文档中对于label 的建议

```bash
CAUTION: Remember that every unique combination of key-value label pairs represents a new time series, which can dramatically increase the amount of data stored. Do not use labels to store dimensions with high cardinality (many different label values), such as user IDs, email addresses, or other unbounded sets of values.
```

如何查看当前的label 分布情况呢，可以使用 prometheus提供的tsdb工具。可以使用命令行查看，也可以在 2.16 版本以上的 prometheus graph 查看

```bash
[work@xxx bin]$ ./tsdb analyze ../data/prometheus/
Block ID: 01E41588AJNGM31SPGHYA3XSXG
Duration: 2h0m0s
Series: 955372
Label names: 301
Postings (unique label pairs): 30757
Postings entries (total label pairs): 10842822
....
```

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/295529ab-e033-4893-9b0a-cb0a3e698b35.jpg?x-oss-process=style/watermark)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/0bd4bdab-2b42-4040-a245-66b9530aab8c.jpg?x-oss-process=style/watermark)

top10 高基数的 metric

```bash
Highest cardinality metric names:
87176 apiserver_request_latencies_bucket
59968 apiserver_response_sizes_bucket
39862 apiserver_request_duration_seconds_bucket
37555 container_tasks_state
....
```

高基数的 label

```bash
Highest cardinality labels:
4271 resource_version
3670 id
3414 name
1857 container_id
1824 __name__
1297 uid
1276 pod
...
```

## 找到最大的 metric 或 job

top10的 metric 数量： 按 metric 名字分

```bash
topk(10, count by (__name__)({__name__=~".+"}))

apiserver_request_latencies_bucket{}  62544
apiserver_response_sizes_bucket{}   44600
```

top10的 metric 数量： 按 job 名字分


```bash
topk(10, count by (__name__, job)({__name__=~".+"}))

{job="master-scrape"}	525667
{job="xxx-kubernetes-cadvisor"}  50817
{job="yyy-kubernetes-cadvisor"}   44261
```

## k8s组件性能指标

除了基础的cadvisor资源监控，还应该对核心组件的 metric 进行采集，包括：

* 10250：kubelet监听端口，包括/stats/summary、metrics、metrics/cadvisor。10250为认证端口，非认证端口用10255
* 10251：kube-scheduler的 metric 端口，本地 127 访问不需要认证，如调度延迟,
* 10252：kube-controller的metric 端口，本地 127 访问不需要认证
* 6443: apiserver，需要证书认证，直接 curl命令为`curl --cacert /etc/kubernetes/pki/ca.pem --cert /etc/kubernetes/pki/admin.pem --key /etc/kubernetes/pki/admin-key.pem https://ip:6443/metrics -k`
* 2379: etcd的 metric 端口，直接 curl命令为：`curl --cacert /etc/etcd/ssl/ca.pem --cert /etc/etcd/ssl/etcd.pem --key /etc/etcd/ssl/etcd-key.pem https://localhost:2379/metrics -k`

docker 指标暴露：

如果要开放 docker进程指标，需要开启实验特性，文件/etc/docker/daemon.json


```bash
{
  "metrics-addr" : "127.0.0.1:9323",
  "experimental" : true
}
```

kube-proxy 指标: 

端口为10249，默认 127开放，可以修改为 hostname 开放，--metrics-bind-address=机器 ip

示例图：
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/4b5a97fb-6064-499a-be48-7ac53e66255d.jpg?x-oss-process=style/watermark)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/45cfe3ed-6e35-4884-bc0b-975d6b5c1de5.jpg?x-oss-process=style/watermark)

## prometheus 重启慢

prometheus重启的时候需要把 wal 中的内容 load 到内存里，保留时间越久、wal 文件越大，重启的实际越长，这个是prometheus的机制，没得办法，因此能 reload 的，就不要重启，重启一定会导致短时间的不可用，而这个时候prometheus高可用就很重要了。

但prometheus 也曾经对启动时间做过优化，在 2.6 版本中对于WAL的 load速度就做过速度的优化，希望重启的时间不超过 [1 分钟](https://www.robustperception.io/optimising-startup-time-of-prometheus-2-6-0-with-pprof)

## 你的应用应该暴露多少指标

当你开发自己的服务的时候，你可能会把一些数据暴露 metric出去，比如特定请求数、goroutine 数等，指标数量多少合适呢？

虽然指标数量和你的应用规模相关，但也有一些[建议(Brian Brazil)](https://www.robustperception.io/how-many-metrics-should-an-application-return),

比如简单的服务如缓存等，类似 pushgateway，大约 120 个指标，prometheus 本身暴露了 700 左右的指标，如果你的应用很大，也尽量不要超过 10000 个指标，需要合理控制你的 label。

## relabel_configs 与metric_relabel_configs

relabel_config发生在采集之前，metric_relabel_configs发生在采集之后，合理搭配可以满足场景的配置

如

```bash
metric_relabel_configs:
  - separator: ;
    regex: instance
    replacement: $1
    action: labeldrop
```

```bash
- source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_endpoints_name,
      __meta_kubernetes_service_annotation_prometheus_io_port]
    separator: ;
    regex: (.+);(.+);(.*)
    target_label: __metrics_path__
    replacement: /api/v1/namespaces/${1}/services/${2}:${3}/proxy/metrics
    action: replace
```

## Prometheus 的预测能力

场景1：你的磁盘剩余空间一直在减少，并且降低的速度比较均匀，你希望知道大概多久之后达到阈值，并希望在某一个时刻报警出来。

场景2：你的pod内存使用率一直升高，你希望知道大概多久之后会到达 limit 值，并在一定时刻报警出来，在被杀掉之前上去排查。

prometheus的deriv和predict_linear方法可以满足这类需求， promtheus 提供了基础的预测能力，基于当前的变化速度，推测一段时间后的值。

以mem_free 为例，最近一小时的 free值一直在下降。

```bash
mem_free仅为举例，实际内存可用以mem_available为准
```
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/467d1419-1757-4130-810c-d00fe0b1169b.jpg?x-oss-process=style/watermark)


deriv函数可以显示指标在一段时间的变化速度

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/56b44830-006e-4d9a-be5b-50e92f5c08bd.jpg?x-oss-process=style/watermark)


predict_linear方法是预测基于这种速度，最后可以达到的值

```bash
predict_linear(mem_free{instanceIP="100.75.155.55"}[1h], 2*3600)/1024/1024
```

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/1967b016-6b0e-4876-98b7-b5c8228c4654.jpg?x-oss-process=style/watermark)

你可以基于设置合理的报警规则，如小于 10 时报警

``` bash
rule: predict_linear(mem_free{instanceIP="100.75.155.55"}[1h], 2*3600)/1024/1024 <10 
```

predict_linear与deriv的关系，含义上约等于，predict_linear稍微准确一些。

```bash
  deriv(mem_free{instanceIP="100.75.155.55"}[1h]) * 2 * 3600
+
  mem_free{instanceIP="100.75.155.55"}[1h]
```

## 错误的高可用设计

有些人提出过这种类型的方案，想提高其扩展性和可用性。
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/84f4f78b-b7fc-48ee-8fb0-e9fe57e8d3b6.jpg?x-oss-process=style/watermark)

应用程序将metric 推到到消息队列如 kafaka，然后经过 exposer消费中转，再被 prometheus 拉取。产生这种方案的原因一般是有历史包袱、复用现有组件、想通过 mq 来提高扩展性。

这种方案有几个问题：

1. 增加了 queue 组件，多了一层依赖，如果 app与 queue 之间连接失败，难道要在app本地缓存监控数据？
2. 抓取时间可能会不同步，延迟的数据将会被标记为陈旧数据，当然你可以通过添加时间戳来标识，但就失去了对陈旧数据的[处理逻辑](https://www.robustperception.io/staleness-and-promql)
3. 扩展性问题：prometheus 适合大量小目标，而不是一个大目标，如果你把所有数据都放在了 exposer 中，那么 prometheus 的单个 job拉取就会成为cpu 瓶颈。这个和 pushgateway 有些类似，没有特别必要的场景，都不是官方建议的方式。
4. 缺少了服务发现和拉取控制，prom只知道一个exposer，不知道具体是哪些 target，不知道他们的 up 时间，无法使用scrape_*等指标做查询，也无法用[scrape_limit](https://www.robustperception.io/using-sample_limit-to-avoid-overload)做限制。

如果你的架构和 prometheus 的设计理念相悖，可能要重新设计一下方案了，否则扩展性和可靠性反而会降低。

## 高可用方案

prometheus 高可用有几种方案：

1. 基本 HA：即两套 prometheus 采集完全一样的数据，外边挂负载均衡
2. HA + 远程存储：除了基础的多副本prometheus，还通过Remote write 写入到远程存储，解决存储持久化问题
3. 联邦集群：即federation，按照功能进行分区，不同的 shard 采集不同的数据，由Global节点来统一存放，解决监控数据规模的问题。
4. 使用thanos 或者victoriametrics，来解决全局查询、多副本数据 join 问题。

就算使用官方建议的多副本 + 联邦，仍然会遇到一些问题:

```bash
官方建议数据做Shard，然后通过federation来实现高可用，
但是边缘节点和Global节点依然是单点，需要自行决定是否每一层都要使用双节点重复采集进行保活。
也就是仍然会有单机瓶颈。

另外部分敏感报警尽量不要通过global节点触发，毕竟从Shard节点到Global节点传输链路的稳定性会影响数据到达的效率，进而导致报警实效降低。

例如服务updown状态，API请求异常这类报警我们都放在shard节点进行报警。
```

本质原因是，prometheus的本地存储没有数据同步能力，要在保证可用性的前提下，再保持数据一致性是比较困难的，基础的 HA proxy 满足不了要求，比如：

* 集群的后端有 A 和 B 两个实例，A 和 B 之间没有数据同步。A 宕机一段时间，丢失了一部分数据，如果负载均衡正常轮询，请求打到A 上时，数据就会异常。
* 如果 A 和 B 的启动时间不同，时钟不同，那么采集同样的数据时间戳也不同，就不是多副本同样数据的概念了
* 就算用了远程存储，A 和 B 不能推送到同一个 tsdb，如果每人推送自己的 tsdb，数据查询走哪边就是问题了。

因此解决方案是在存储、查询两个角度上保证数据的一致:

* 存储角度：如果使用 remote write 远程存储， A 和 B后面可以都加一个 adapter，adapter做选主逻辑，只有一份数据能推送到 tsdb，这样可以保证一个异常，另一个也能推送成功，数据不丢，同时远程存储只有一份，是共享数据。方案可以参考[这篇文章](https://blog.timescale.com/blog/prometheus-ha-postgresql-8de68d19b6f5)
* 查询角度：上边的方案实现很复杂且有一定风险，因此现在的大多数方案在查询层面做文章，比如thanos 或者victoriametrics，仍然是两份数据，但是查询时做数据去重和join。只是 thanos是通过 sidecar 把数据放在对象存储，victoriametrics是把数据remote write 到自己的 server 实例，但查询层 thanos-query 和victor的 promxy的逻辑基本一致。

我们采用了thanos来支持多地域监控数据，具体方案可以看[这篇文章](http://www.xuyasong.com/?p=1925)

## 关于日志

k8s 中的日志一般指是容器标准输出 + 容器内日志，方案基本是采用 fluentd/fluent-bit/filebeat等采集推送到 es，但还有一种是日志转 metric，如解析特定字符串出现次数，nginx 日志得到qps指标 等，这里可以采用 grok或者 mtail，以 exporter的形式提供 metric 给 prometheus

## 参考

* 美团 cat:https://tech.meituan.com/2018/11/01/cat-in-depth-java-application-monitoring.html
* 360:https://dbaplus.cn/news-134-1462-1.html
* 携程：https://www.infoq.cn/article/P3A5EuKl6jowO9v4_ty1
* 分类：https://zhuanlan.zhihu.com/p/60791449
* http://download.xuliangwei.com/jiankong.html#0-%E7%9B%91%E6%8E%A7%E7%9B%AE%E6%A0%87
* https://aleiwu.com/post/prometheus-bp/
* https://www.infoq.cn/article/1AofGj2SvqrjW3BKwXlN
* http://bos.itdks.com/e8792eb0577b4105b4c9d19eb0dd1892.pdf
* https://www.infoq.cn/article/ff46l7LxcWAX698zpzB2
* https://www.robustperception.io/putting-queues-in-front-of-prometheus-for-reliability
* https://www.slideshare.net/cadaam/devops-taiwan-monitor-tools-prometheus
* https://www.robustperception.io/how-much-disk-space-do-prometheus-blocks-use
* https://www.imooc.com/article/296613
* https://dzone.com/articles/what-is-high-cardinality
* https://www.robustperception.io/cardinality-is-key
* https://www.robustperception.io/using-tsdb-analyze-to-investigate-churn-and-cardinality
* https://blog.timescale.com/blog/prometheus-ha-postgresql-8de68d19b6f5/
* https://asktug.com/t/topic/2618


本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)


