
## 背景

在[prometheus 使用心得](http://www.xuyasong.com/?p=1921)文章中有简单提到prometheus 的高可用方案，尝试了联邦、remote write 之后，我们最终选择了 thanos 作为监控配套组件，利用其全局视图来管理我们的多地域、200+集群的监控数据。本文主要介绍 thanos 的一些组件使用和心得体会。

prometheus官方的高可用有几种方案：

1. HA：即两套 prometheus 采集完全一样的数据，外边挂负载均衡
2. HA + 远程存储：除了基础的多副本prometheus，还通过Remote write 写入到远程存储，解决存储持久化问题
3. 联邦集群：即federation，按照功能进行分区，不同的 shard 采集不同的数据，由Global节点来统一存放，解决监控数据规模的问题。

使用官方建议的多副本 + 联邦仍然会遇到一些问题，本质原因是prometheus的本地存储没有数据同步能力，要在保证可用性的前提下再保持数据一致性是比较困难的，基本的多副本 proxy 满足不了要求，比如：

* prometheus集群的后端有 A 和 B 两个实例，A 和 B 之间没有数据同步。A 宕机一段时间，丢失了一部分数据，如果负载均衡正常轮询，请求打到A 上时，数据就会异常。
* 如果 A 和 B 的启动时间不同，时钟不同，那么采集同样的数据时间戳也不同，就多副本的数据不相同
* 就算用了远程存储，A 和 B 不能推送到同一个 tsdb，如果每人推送自己的 tsdb，数据查询走哪边就是问题
* 官方建议数据做Shard，然后通过federation来实现高可用，但是边缘节点和Global节点依然是单点，需要自行决定是否每一层都要使用双节点重复采集进行保活。也就是仍然会有单机瓶颈。
* 另外部分敏感报警尽量不要通过global节点触发，毕竟从Shard节点到Global节点传输链路的稳定性会影响数据到达的效率，进而导致报警实效降低。

目前大多数的 prometheus 的集群方案是在存储、查询两个角度上保证数据的一致:

* 存储角度：如果使用 remote write 远程存储， A 和 B后面可以都加一个 adapter，adapter做选主逻辑，只有一份数据能推送到 tsdb，这样可以保证一个异常，另一个也能推送成功，数据不丢，同时远程存储只有一份，是共享数据。方案可以参考[这篇文章](https://blog.timescale.com/blog/prometheus-ha-postgresql-8de68d19b6f5)
* 存储角度：仍然使用 remote write 远程存储，但是 A 和 B 分别写入 tsdb1 和 tsdb2 两个时序数据库，利用sync的方式在 tsdb1 和2 之前做数据同步，保证数据是全量的。
* 查询角度：上边的方案需要自己实现，有侵入性且有一定风险，因此大多数开源方案是在查询层面做文章，比如thanos 或者victoriametrics，仍然是两份数据，但是查询时做数据去重和join。只是 thanos是通过 sidecar 把数据放在对象存储，victoriametrics是把数据remote write 到自己的 server 实例，但查询层 thanos-query 和victor的 promxy的逻辑基本一致，都是为全局视图服务

## 实际需求

随着我们的集群规模越来越大，监控数据的种类和数量也越来越多：如master/node 机器监控、进程监控、4 大核心组件的性能监控，pod 资源监控、kube-stats-metrics、k8s events监控、插件监控等等。除了解决上面的高可用问题，我们还希望基于 prometheus 构建全局视图，主要需求有：

* 长期存储：1 个月左右的数据存储，每天可能新增几十G，希望存储的维护成本足够小，有容灾和迁移。考虑过使用 influxdb，但influxdb没有现成的集群方案，且需要人力维护。最好是存放在云上的 tsdb 或者对象存储、文件存储上。
* 无限拓展：我们有300+集群，几千节点，上万个服务，单机prometheus无法满足，且为了隔离性，最好按功能做 shard，如 master 组件性能监控与 pod 资源等业务监控分开、主机监控与日志监控也分开。或者按租户、业务类型分开（实时业务、离线业务）。
* 全局视图：按类型分开之后，虽然数据分散了，但监控视图需要整合在一起，一个 grafana 里 n个面板就可以看到所有地域+集群+pod 的监控数据，操作更方便，不用多个 grafana 切来切去，或者 grafana中多个 datasource 切来切去。
* 无侵入性：不要对已有的 prometheus 做过多的修改，因为 prometheus 是开源项目，版本也在快速迭代，我们最早使用过 1.x，可1.x 和 2.x的版本升级也就不到一年时间，2.x 的存储结构查询速度等都有了明显提升，1.x 已经没人使用了。因此我们需要跟着社区走，及时迭代新版本。因此不能对 prometheus 本身代码做修改，最好做封装，对最上层用户透明。

在调研了大量的开源方案(cortex/thanos/victoria/..)和商业产品之后，我们选择了 thanos，准确的说，thanos只是监控套件，与 原生prometheus 结合，满足了长期存储+ 无限拓展 + 全局视图 + 无侵入性的需求。


## thanos 架构

thanos 的默认模式：sidecar 方式

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/af3a7828-826b-42c7-9798-941867c67897.jpg?x-oss-process=style/watermark)
除了 这个sidecar 方式，thanos还有一种不太常用的receive 模式，后面会提到。

Thanos是一组组件，在[官网](https://thanos.io/)上可以看到包括：

* Bucket
* Check
* Compactor
* Query
* Rule
* Sidecar
* Store

除了官方提到的这些，其实还有：

* receive
* downsample

看起来组件很多，但其实部署时二进制只有一个，非常方便。只是搭配不同的参数实现不同的功能，如 query 组件就是 ./thanos query，sidecar 组件就是./thanos sidecar，组件all in one，代码只有一份，体积很小。

其实核心的sidecar+query就已经可以运行，其他的组件只是为了实现更多的功能

最新版 thanos 在 这里下载[release](https://github.com/thanos-io/thanos/releases)，对于 thanos这种仍然在修bug、迭代功能的软件，有新版本就不要用旧的。


## 组件与配置

下面会介绍如何组合thanos组件，来快速实现你的 prometheus 高可用，因为是快速介绍，和官方的 [quick start](https://thanos.io/quick-tutorial.md/)有一部分雷同，且本文截止2020.1 月的版本，不知道以后会thanos 会迭代成什么样子


### 第 1 步：确认已有的 prometheus

thanos 是无侵入的，只是上层套件，因此你还是需要部署你的 prometheus，这里不再赘述，默认你已经有一个单机的 prometheus在运行，可以是 pod 也可以是主机部署，取决于你的运行环境，我们是在 k8s 集群外，因此是主机部署。prometheus采集的是地域A的监控数据。你的 prometheus配置可以是：

启动配置：

```bash
"./prometheus
--config.file=prometheus.yml \
--log.level=info \
--storage.tsdb.path=data/prometheus \
--web.listen-address='0.0.0.0:9090' \
--storage.tsdb.max-block-duration=2h \
--storage.tsdb.min-block-duration=2h \
--storage.tsdb.wal-compression \
--storage.tsdb.retention.time=2h \
--web.enable-lifecycle"
```

web.enable-lifecycle一定要开，用于热加载reload你的配置，retention保留 2 小时，prometheus 默认 2 小时会生成一个 block，thanos 会把这个 block 上传到对象存储。

采集配置：prometheus.yml

```yaml
global:
  scrape_interval:     60s
  evaluation_interval: 60s
  external_labels:
     region: 'A'
     replica: 0

rule_files:
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['0.0.0.0:9090']

  - job_name: 'demo-scrape'
    metrics_path: '/metrics'
    params:
    ...
```

这里需要声明external_labels，标注你的地域。如果你是多副本运行，需要声明你的副本标识，如 0号，1，2 三个副本采集一模一样的数据，另外2个 prometheus就可以同时运行，只是replica值不同而已。这里的配置和官方的 federation差不多。

对 prometheus 的要求：

* 2.2.1版本以上
* 声明你的external_labels
* 启用--web.enable-admin-api
* 启用--web.enable-lifecycle


### 第 2 步：部署 sidecar 组件

关键的步骤来了，最核心莫过于 sidecar组件。sidecar是 k8s 中的一种[模式](https://www.kubernetes.org.cn/6405.html)

Sidecar 组件作为 Prometheus server pod 的 sidecar 容器，与 Prometheus server 部署于同一个 pod 中。 他有两个作用：

1. 它使用Prometheus的remote read API，实现了Thanos的Store API。这使后面要介绍的Query 组件可以将Prometheus服务器视为时间序列数据的另一个来源，而无需直接与Prometheus API交互（这就是 sidecar 的拦截作用）
2. 可选配置：在Prometheus每2小时生成一次TSDB块时，Sidecar将TSDB块上载到对象存储桶中。这使得Prometheus服务器可以以较低的保留时间运行，同时使历史数据持久且可通过对象存储查询。

当然，这不意味着Prometheus可以是完全无状态的，因为如果它崩溃并重新启动，您将丢失2个小时的指标，不过如果你的 prometheus 也是多副本，可以减少这2h 数据的风险。

sidecar配置：

```bash
./thanos sidecar \
--prometheus.url="http://localhost:8090" \
--objstore.config-file=./conf/bos.yaml \
--tsdb.path=/home/work/opdir/monitor/prometheus/data/prometheus/
"
```

配置很简单，只需要声明prometheus.url和数据地址即可。objstore.config-file是可选项。如果你要把数据存放在对象存储（这也是推荐做法），就配置下对象存储的账号信息。

thanos 默认支持谷歌云/AWS等，以 谷歌云为例，配置如下：

```yaml
type: GCS
config:
  bucket: ""
  service_account: ""
```

因为thanos默认还不支持我们的云存储，因此我们在 thanos代码中加入了相应的实现，并向官方提交了 pr。

需要注意的是：别忘了为你的另外两个副本 1号 和 2号prometheus都搭配一个 sidecar。如果是 pod运行可以加一个 container，127 访问，如果是主机部署，指定prometheus端口就行。

另外 sidecar是无状态的，也可以多副本，多个 sidecar 可以访问一份 prometheus 数据，保证 sidecar本身的拓展性，不过如果是 pod 运行也就没有这个必要了，sidecar和prometheus 同生共死就行了。

sidecar 会读取prometheus 每个 block 中的 meta.json信息，然后扩展这个 json 文件，加入了 Thanos所特有的 metadata 信息。而后上传到块存储上。上传后写入thanos.shipper.json 中


### 第 3 步：部署 query 组件

sidecar 部署完成了，也有了 3 个一样的数据副本，这个时候如果想直接展示数据，可以安装 query 组件

Query组件（也称为“查询”）实现了Prometheus 的HTTP v1 API，可以像 prometheus 的 graph一样，通过PromQL查询Thanos集群中的数据。

简而言之，sidecar暴露了StoreAPI，Query从多个StoreAPI中收集数据，查询并返回结果。Query是完全无状态的，可以水平扩展。

配置：

```bash
"
./thanos query \
--http-address="0.0.0.0:8090" \
--store=relica0:10901 \
--store=relica1:10901 \
--store=relica2:10901 \
--store=127.0.0.1:19914 \
"
```

store 参数代表的就是刚刚启动的 sidecar 组件，启动了 3 份，就可以配置三个relica0、relica1、relica2，10901 是 sidecar 的默认端口。

http-address 代表 query 组件本身的端口，因为他是个 web 服务，启动后，页面是这样的：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/4410b1ab-7997-46f1-839a-4db59b4d763d.jpg?x-oss-process=style/watermark)


和 prometheus 几乎一样对吧，有了这个页面你就不需要关心最初的 prometheus 了，可以放在这里查询。

点击 store，可以看到对接了哪些 sidecar。


![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/7b2a286f-9caf-4d01-b4e8-016cea5db820.jpg?x-oss-process=style/watermark)

query 页面有两个勾选框，含义是：

* deduplication：是否去重。默认勾选代表去重，同样的数据只会出现一条，否则 replica0 和 1、2 完全相同的数据会查出来 3 条。
* partial response：是否允许部分响应，默认允许，这里有一致性的折中，比如 0、1、2 三副本有一个挂掉或者超时了，查询时就会有一个没有响应，如果允许返回用户剩下的 2 份，数据就没有很强的一致性，但因为一个超时就完全不返回，就丢掉了可用性，因此默认允许部分响应。


### 第 4 步：部署 store gateway 组件


你可能注意到了，在第 3 步里，./thanos query有一条--store是 xxx:19914，并不是一直提到的 3 副本，这个 19914 就是接下来要说的store gateway组件。

在第 2 步的 sidecar 配置中，如果你配置了对象存储objstore.config-file，你的数据就会定时上传到bucket 中，本地只留 2 小时，那么要想查询 2 小时前的数据怎么办呢？数据不被 prometheus 控制了，应该如何从 bucket 中拿回来，并提供一模一样的查询呢？
 
Store gateway 组件：Store gateway 主要与对象存储交互，从对象存储获取已经持久化的数据。与sidecar一样，Store gateway也实现了store api，query 组可以从 store gateway 查询历史数据。
 
配置如下：
 
```bash
./thanos store \
--data-dir=./thanos-store-gateway/tmp/store \
--objstore.config-file=./thanos-store-gateway/conf/bos.yaml \
--http-address=0.0.0.0:19904 \
--grpc-address=0.0.0.0:19914 \
--index-cache-size=250MB \
--sync-block-duration=5m \
--min-time=-2w \
--max-time=-1h \
```

grpc-address就是store api暴露的端口，也就是query 中--store是 xxx:19914的配置。

因为Store gateway需要从网络上拉取大量历史数据加载到内存，因此会大量消耗 cpu 和内存，这个组件也是 thanos 面世时被质疑过的组件，不过当前的性能还算可以，遇到的一些问题后面会提到。
 
Store gateway也可以无限拓展，拉取同一份 bucket 数据。

放个示意图，一个 thanos 副本，挂了多个地域的 store 组件
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/d4c9e5cd-7763-461f-8c64-cfa4e1cb2b89.jpg?x-oss-process=style/watermark)


其中一个地域的数据统计：

查询一个月历史数据速度还可以，主要是数据持久化没有运维压力，随意扩展，成本低。

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/11c0871c-4767-4501-9ddc-a362334ed36f.jpg?x-oss-process=style/watermark)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/0bf78b72-b72b-4a45-ba00-1c0885260337.jpg?x-oss-process=style/watermark)


到这里，thanos 的基本使用就结束了，至于 compact 压缩和 bucket 校验，不是核心功能，compact我们只是简单部署了一下，rule组件我们没有使用，就不做介绍了。

#### 5.查看数据

有了多地域多副本的数据，就可以结合 grafana 做全局视图了，比如：

按地域和集群查看 etcd 的性能指标：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/aa1270a9-7190-4dcf-8665-b53742b9c550.jpg?x-oss-process=style/watermark)

按地域、集群、机器查看核心组件监控，如多副本 master 机器上的各种性能

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/e2b10a46-2580-4f8a-89a0-c59b70777c9d.jpg?x-oss-process=style/watermark)

数据聚合在一起之后，可以将所有视图都集中展示，比如还有这些面板：

* 机器监控：node-exporter、process-exporter
* pod 资源使用: cadvisor
* docker、kube-proxy、kubelet 监控
* scheduler、controller-manager、etcd、apiserver 监控
* kube-state-metrics 元信息
* k8s events 
* mtail 等日志监控


## Receive 模式

前面提到的所有组件都是基于 sidecar 模式配置的，但thanos还有一种Receive模式，不太常用，只是在[Proposals中出现](https://thanos.io/proposals/201812_thanos-remote-receive.md)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/9df5c0d5-b15b-4f52-92ac-86bd9a347b56.jpg?x-oss-process=style/watermark)

因为一些网络限制，我们之前尝试过Receive方案，这里可以描述下Receive的使用场景：

1. sidecar 模式有一个缺点：就是2 小时内的数据仍然需要通过 sidecar->prometheus来获取，也就是仍然依赖 prometheus，并不是完全的数据在外部存储。如果你的网络只允许你查询特定的存储数据，无法达到集群内的prometheus，那这 2 小时的数据就丢失了，而 Receive模式采用了remote write 就没有所谓的 2 小时 block 的问题了。
2. sidecar 模式对网络连通性是有要求的，如果你是多租户环境或者是云厂商，对象存储（历史数据）query 组件一般在控制面，方便做权限校验和接口服务封装，而 sidecar 和 prometheus却在集群内，也就是用户侧。控制面和用户侧的网络有时候会有限制，是不通的，这个时候会有一些限制导致你无法使用 sidecar
3. 租户和控制面隔离，和第2 条类似，希望数据完全存在控制面，我一直觉得Receive就是为了云厂商服务的。。

不过Receive毕竟不是默认方案，如果不是特别需要，还是用默认的 sidecar 为好

## 一些问题

### prometheus 压缩

 压缩：官方文档有提到，使用sidecar时，需要将 prometheus 的--storage.tsdb.min-block-duration 和 --storage.tsdb.max-block-duration，这两个值设置为2h，两个参数相等才能保证prometheus关闭了本地压缩，其实这两个参数在 prometheus -help 中并没有体现，prometheus 作者也说明这只是为了开发测试才用的参数，不建议用户修改。而 thanos 要求关闭压缩是因为 prometheus 默认会以2，2*5，2*5*5的周期进行压缩，如果不关闭，可能会导致 thanos 刚要上传一个 block，这个 block 却被压缩中，导致上传失败。

不过你也不必担心，因为在 sidecar 启动时，会检查这两个参数，如果不合适，sidecar会启动失败
![43a131c689d9fedba5a7844363876ee7](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/4d174e53-959a-49da-87ae-c53563b57f4c.jpg?x-oss-process=style/watermark)


### store-gateway

store-gateway： store 组件资源消耗是最大的，毕竟他要拉取远程数据，并加载到本地供查询，如果你想控制历史数据和缓存周期，可以修改相应的配置，如

```bash
--index-cache-size=250MB \
--sync-block-duration=5m \ 
--min-time=-2w \ 最大查询 1 周
--max-time=-1h \
```

store-gateway 默认支持索引缓存，来加快tsdb 块的查找速度，但有时候启动会占用了大量的内存，在 0.11.0之后的版本做了修复，可以查看这个[issue](https://github.com/thanos-io/thanos/issues/448)

Prometheus 2.0 已经对存储层进行了优化。例如按照时间和指标名字，连续的尽量放在一起。而 store gateway可以获取存储文件的结构，因此可以很好的将指标存储的请求翻译为最少的 object storage 请求。对于那种大查询，一次可以拿成百上千个 chunks 数据。

而在 store 的本地，只有 index 数据是放入 cache的，chunk 数据虽然也可以，但是就要大几个数量级了。目前，从对象存储获取 chunk 数据只有很小的延时，因此也没什么动力去将 chunk 数据给 cache起来，毕竟这个对资源的需求很大。

store-gateway中的数据：
![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/c0569292-fe73-4867-80f8-11889ae1249b.jpg?x-oss-process=style/watermark)

每个文件夹中其实是一个个的索引文件index.cache.json

### compactor组件

prometheus数据越来越多，查询一定会越来越慢，thanos提供了一个compactor组件来处理，他有两个功能，

* 一个是做压缩，就是把旧的数据不断的合并。
* 另外一个是降采样，他会把存储的数据，按照一定的时间，算出最大，最小等值，会根据查询的间隔，进行控制，返回采样的数据，而不是真实的点，在查询特别长的时间的数据的时候，看的主要是趋势，精度是可以选择下降的。
* 注意的是compactor并不会减少磁盘占用，反而会增加磁盘占用（做了更高维度的聚合）。

通过以上的方式，有效了优化查询，但是并不是万能的。因为业务数据总在增长，这时候可能要考虑业务拆分了，我们需要对业务有一定的估算，例如不同的业务存储在不同bucket里（需要改造或者多部署几个 sidecar）。例如有5个bucket，再准备5个store gateway进行代理查询。减少单个 store 数据过大的问题。

第二个方案是时间切片，也就是就是上面提到的store gateway可以选择查询多长时间的数据。支持两种表达，一种是基于相对时间的，例如--max-time 3d前到5d前的。一种是基于绝对时间的，19年3月1号到19年5月1号。例如想查询3个月的数据，一个store代理一个月的数据，那么就需要3个store来合作。

### query 的去重

query组件启动时，默认会根据query.replica-label字段做重复数据的去重，你也可以在页面上勾选**deduplication** 来决定。query 的结果会根据你的query.replica-label的 label选择副本中的一个进行展示。可如果 0，1，2 三个副本都返回了数据，且值不同，query 会选择哪一个展示呢？

thanos会基于打分机制，选择更为稳定的 replica 数据, 具体逻辑在：https://github.com/thanos-io/thanos/blob/55cb8ca38b3539381dc6a781e637df15c694e50a/pkg/query/iter.go

## 参考

* https://thanos.io/
* https://www.percona.com/blog/2018/09/20/prometheus-2-times-series-storage-performance-analyses/
* https://qianyongchao.blog/2019/01/03/prometheus-thanos-design-%E4%BB%8B%E7%BB%8D/
* https://github.com/thanos-io/thanos/issues/405
* https://katacoda.com/bwplotka/courses/thanos
* https://medium.com/faun/comparing-thanos-to-victoriametrics-cluster-b193bea1683
* https://www.youtube.com/watch?v=qQN0N14HXPM


本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)


