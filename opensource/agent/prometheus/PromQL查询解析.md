
### 一. 概述

Prometheus除了存储数据外，还提供了一种强大的功能表达式语言 PromQL，允许用户实时选择和汇聚时间序列数据。

表达式的结果可以在浏览器中显示为图形，也可以显示为表格数据，或者由外部系统通过 HTTP API 调用。通过PromQL用户可以非常方便地查询监控数据，或者利用表达式进行告警配置

如：k8s中的node在线率：sum(kube_node_status_condition{condition="Ready", status="true"}) / sum(kube_node_info) *100 

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15516050414198.jpg)

### Metric类型

关于时间序列存储，可以参考：https://www.infoq.cn/article/database-timestamp-01

Prometheus会将所有采集到的样本数据以时间序列（time-series）的方式保存在内存数据库TSDB中，并且定时保存到硬盘上。time-series是按照时间戳和值的序列顺序存放的，我们称之为向量(vector)。每条time-series通过指标名称(metrics name)和一组标签集(labelset)命名。

在time-series中的每一个点称为一个样本（sample），样本由以下三部分组成：

* 指标(metric)：metric name和描述当前样本特征的labelsets;
* 时间戳(timestamp)：一个精确到毫秒的时间戳;
* 样本值(value)： 一个folat64的浮点型数据表示当前样本的值。


如某一时刻的node_cpu指标为459.71

```text

node_cpu{app="node-exporter",cpu="cpu0",instance="192.168.0.4:9100",job="kubernetes-service-endpoints",kubernetes_name="node-exporter",kubernetes_namespace="kube-system",mode="guest"}	 459.71
```

Prometheus定义了4中不同的指标类型(metric type):

* Counter 计数器

```text

计数器，只增不减，如http_requests_total请求总数

例如，通过rate()函数获取HTTP请求量的增长率：
rate(http_requests_total[5m])
```
 

* Gauge 仪表盘

```text

当前状态，可增可减。如kube_pod_status_ready当前pod可用数
可以获取样本在一段时间返回内的变化情况,如：
delta(kube_pod_status_ready[2h])

```

* Histogram 直方图


```text

Histogram 由 <basename>_bucket{le="<upper inclusive bound>"}，<basename>_bucket{le="+Inf"}, <basename>_sum，<basename>_count 组成，主要用于表示一段时间范围内对数据进行采样（通常是请求持续时间或响应大小），并能够对其指定区间以及总数进行统计，通常它采集的数据展示为直方图。

例如 Prometheus server 中 prometheus_local_storage_series_chunks_persisted, 表示 Prometheus 中每个时序需要存储的 chunks 数量，我们可以用它计算待持久化的数据的分位数。
```

* Summary 摘要

```text

Summary 和 Histogram 类似，由 <basename>{quantile="<φ>"}，<basename>_sum，<basename>_count 组成，主要用于表示一段时间内数据采样结果（通常是请求持续时间或响应大小），它直接存储了 quantile 数据，而不是根据统计区间计算出来的。

例如 Prometheus server 中 prometheus_target_interval_length_seconds。

Histogram 需要通过 <basename>_bucket 计算 quantile, 而 Summary 直接存储了 quantile 的值。

```

### 基础查询

PromQL是Prometheus内置的数据查询语言，其提供对时间序列数据丰富的查询，聚合以及逻辑运算能力的支持。

如http_requests_total指标

你可以通过附加一组标签，并用{}括起来，来进一步筛选这些时间序列。下面这个例子只选择有http_requests_total名称的、有prometheus工作标签的、有canary组标签的时间序列：

```text

http_requests_total{job="prometheus",group="canary"}

```
  
如果条件为空，可以写为：http_requests_total{}

另外，也可以也可以将标签值反向匹配，或者对正则表达式匹配标签值。如操作符：
  
```text

  =：选择正好相等的字符串标签
  !=：选择不相等的字符串标签
  =~：选择匹配正则表达式的标签（或子标签）
  !~：选择不匹配正则表达式的标签（或子标签）
  
```

### 范围查询

类似`http_requests_total{job="prometheus",group="canary"}`的方式，得到的是瞬时值，如果想得到一定范围内的值，可以使用范围查询

时间范围通过时间范围选择器[]进行定义。例如，通过以下表达式可以选择最近5分钟内的所有样本数据，如：http_request_total{}[5m]

除了分钟，支持的单位有：

* s - 秒
* m - 分钟
* h - 小时
* d - 天
* w - 周
* y - 年

### 偏移查询

如：查询http_requests_total在当前时刻的一周的速率：

```text
rate(http_requests_total{} offset 1w)
```

偏移修饰符允许更改查询中单个即时向量和范围向量的时间偏移量，例如，以下表达式返回相对于当前查询时间5分钟前的http_requests_total值：

`http_requests_total offset 5m`

等价于

`http_requests_total{job="prometheus"}[5m]`

请注意，偏移量修饰符始终需要跟随选择器，即以下是正确的：

  `sum(http_requests_total{method="GET"} offset 5m) // GOOD.`

下面是错误的:

  `sum(http_requests_total{method="GET"}) offset 5m // INVALID.`


### 操作符

Prometheus 的查询语言支持基本的逻辑运算和算术运算

##### 二元算术运算：

* + 加法
* - 减法
* * 乘法
* / 除法
* % 模
* ^ 幂等

运算中用到的基础数据类型：

* 瞬时向量（Instant vector） - 一组时间序列，每个时间序列包含单个样本，它们共享相同的时间戳。也就是说，表达式的返回值中只会包含该时间序列中的最新的一个样本值。而相应的这样的表达式称之为瞬时向量表达式。
* 区间向量（Range vector） - 一组时间序列，每个时间序列包含一段时间范围内的样本数据。
* 标量（Scalar） - 一个浮点型的数据值。
* 字符串（String） - 一个简单的字符串值。

二元运算操作符支持 scalar/scalar(标量/标量)、vector/scalar(向量/标量)、和 vector/vector(向量/向量) 之间的操作。

在两个标量之间进行数学运算，得到的结果也是标量。

例如，如果我们想根据 node_disk_bytes_written 和 node_disk_bytes_read 获取主机磁盘IO的总量，可以使用如下表达式：

`node_disk_bytes_written + node_disk_bytes_read
`

或者node的内存数GB

```
node_memory_free_bytes_total / (1024 * 1024)
```


##### 布尔运算

* == (相等)
* != (不相等)
* > (大于)
* < (小于)
* >= (大于等于)
* <= (小于等于)

如：获取http_requests_total请求总数是否超过10000，返回0和1，1则报警

```text

http_requests_total > 10000 # 结果为 true 或 false
http_requests_total > bool 10000 # 结果为 1 或 0

```

##### 集合运算

* and (并且)
* or (或者)
* unless (排除)

##### 优先级

四则运算有优先级，promql的复杂运算也有优先级

例如，查询主机的CPU使用率，可以使用表达式：

```text

100 * (1 - avg (irate(node_cpu{mode='idle'}[5m])) by(job) )

```

其中irate是PromQL中的内置函数，用于计算区间向量中时间序列每秒的即时增长率
在PromQL操作符中优先级由高到低依次为：

1. ^
2. *, /, %
3. +, -
4. ==, !=, <=, <, >=, >
5. and, unless
6. or


##### 匹配模式（联合查询）

与数据库中的join类似，promsql有两种典型的匹配查询：

* 一对一（one-to-one）
* 多对一（many-to-one）或一对多（one-to-many）

例如当存在样本：

```text

method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120

```

使用 PromQL 表达式：


```text

method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m

```

该表达式会返回在过去 5 分钟内，HTTP 请求状态码为 500 的在所有请求中的比例。如果没有使用 ignoring(code)，操作符两边表达式返回的瞬时向量中将找不到任何一个标签完全相同的匹配项。

因此结果如下：

{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120

同时由于 method 为 put 和 del 的样本找不到匹配项，因此不会出现在结果当中。


**多对一模式**

例如，使用表达式：


```text

method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m

```

该表达式中，左向量 method_code:http_errors:rate5m 包含两个标签 method 和 code。而右向量 method:http_requests:rate5m 中只包含一个标签 method，因此匹配时需要使用 ignoring 限定匹配的标签为 code。 

在限定匹配标签后，右向量中的元素可能匹配到多个左向量中的元素 因此该表达式的匹配模式为多对一，需要使用 group 修饰符 group_left 指定左向量具有更好的基数。

最终的运算结果如下：

{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120

```text

提醒：group 修饰符只能在比较和数学运算符中使用。在逻辑运算 and，unless 和 or 操作中默认与右向量中的所有元素进行匹配。

```

### 聚合操作

Prometheus 还提供了下列内置的聚合操作符，这些操作符作用域瞬时向量。可以将瞬时表达式返回的样本数据进行聚合，形成一个具有较少样本值的新的时间序列。

* sum (求和)
* min (最小值)
* max (最大值)
* avg (平均值)
* stddev (标准差)
* stdvar (标准差异)
* count (计数)
* count_values (对 value 进行计数)
* bottomk (样本值最小的 k 个元素)
* topk (样本值最大的k个元素)
* quantile (分布统计)


这些操作符被用于聚合所有标签维度，或者通过 without 或者 by 子语句来保留不同的维度。

* without 用于从计算结果中移除列举的标签，而保留其它标签。
* by 则正好相反，结果向量中只保留列出的标签，其余标签则移除。

通过 without 和 by 可以按照样本的问题对数据进行聚合。


例如：

如果指标 http_requests_total 的时间序列的标签集为 application, instance, 和 group，我们可以通过以下方式计算所有 instance 中每个 application 和 group 的请求总量：

`sum(http_requests_total) without (instance)`

等价于

`
 sum(http_requests_total) by (application, group)
`

如果只需要计算整个应用的 HTTP 请求总量，可以直接使用表达式：

`sum(http_requests_total)
`

count_values 用于时间序列中每一个样本值出现的次数。count_values 会为每一个唯一的样本值输出一个时间序列，并且每一个时间序列包含一个额外的标签。

这个标签的名字由聚合参数指定，同时这个标签值是唯一的样本值。

例如要计算运行每个构建版本的二进制文件的数量：

```text

count_values("version", build_version)
返回结果如下：

{count="641"}   1
{count="3226"}  2
{count="644"}   4

```

**topk 和 bottomk **

则用于对样本值进行排序，返回当前样本值前 n 位，或者后 n 位的时间序列。

获取 HTTP 请求数前 5 位的时序样本数据，可以使用表达式：

```text
topk(5, http_requests_total)

```

quantile 用于计算当前样本数据值的分布情况 quantile(φ, express) ，其中 0 ≤ φ ≤ 1

```text

例如，当 φ 为 0.5 时，即表示找到当前样本数据中的中位数：

quantile(0.5, http_requests_total)
返回结果如下：

{}   656

```

### 内置函数

Prometheus 提供了其它大量的内置函数，可以对时序数据进行丰富的处理。如上文提到的irate

```text

100 * (1 - avg (irate(node_cpu{mode='idle'}[5m])) by(job) )

```

常用的有：

两分钟内的平均CPU使用率： 

`rate(node_cpu[2m])`

和

`irate(node_cpu[2m])`

```text

需要注意的是使用rate或者increase函数去计算样本的平均增长速率，容易陷入“长尾问题”当中，

其无法反应在时间窗口内样本数据的突发变化。 

例如，对于主机而言在2分钟的时间窗口内，可能在某一个由于访问量或者其它问题导致CPU占用100%的情况，

但是通过计算在时间窗口内的平均增长率却无法反应出该问题。

为了解决该问题，PromQL提供了另外一个灵敏度更高的函数irate(v range-vector)。

irate同样用于计算区间向量的计算率，但是其反应出的是瞬时增长率。

irate函数是通过区间向量中最后两个两本数据来计算区间向量的增长速率。

这种方式可以避免在时间窗口范围内的“长尾问题”，并且体现出更好的灵敏度，通过irate函数绘制的图标能够更好的反应样本数据的瞬时变化状态。
```

irate函数相比于rate函数提供了更高的灵敏度，不过当需要分析长期趋势或者在告警规则中，irate的这种灵敏度反而容易造成干扰。

因此在长期趋势分析或者告警中更推荐使用rate函数。


完整的函数列表为：

* abs()
* absent()
* ceil()
* changes()
* clamp_max()
* clamp_min()
* day_of_month()
* day_of_week()
* days_in_month()
* delta()
* deriv()
* exp()
* floor()
* histogram_quantile()
* holt_winters()
* hour()
* idelta()
* increase()
* irate()
* label_join()
* label_replace()
* ln()
* log2()
* log10()
* minute()
* month()
* predict_linear()
* rate()
* resets()
* round()
* scalar()
* sort()
* sort_desc()
* sqrt()
* time()
* timestamp()
* vector()
* year()
* <aggregation>_over_time()

### API访问


Prometheus当前稳定的HTTP API可以通过/api/v1访问

错误状态码：

* 404 Bad Request：当参数错误或者缺失时。
* 422 Unprocessable Entity 当表达式无法执行时。
* 503 Service Unavailiable 当请求超时或者被中断时。

所有的API请求均使用以下的JSON格式：

```json

{
  "status": "success" | "error",
  "data": <data>,

  // 为error时，有如下报错信息
  "errorType": "<string>",
  "error": "<string>"
}

```

通过HTTP API我们可以分别通过/api/v1/query和/api/v1/query_range查询PromQL表达式当前或者一定时间范围内的计算结果。

##### 瞬时数据查询

URL请求参数：

* query=：PromQL表达式。
* time=：用于指定用于计算PromQL的时间戳。可选参数，默认情况下使用当前系统时间。
* timeout=：超时设置。可选参数，默认情况下使用-query,timeout的全局设置。

```bash
$ curl 'http://localhost:9090/api/v1/query?query=up&time=2015-07-01T20:10:51.781Z'
```

返回：

```json
{
   "status" : "success",
   "data" : {
      "resultType" : "vector",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "value": [ 1435781451.781, "1" ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9100"
            },
            "value" : [ 1435781451.781, "0" ]
         }
      ]
   }
}

```

##### 区间查询

URL请求参数：

* query=: PromQL表达式。
* start=: 起始时间。
* end=: 结束时间。
* step=: 查询步长。
* timeout=: 超时设置。可选参数，默认情况下使用-query,timeout的全局设置。


```bash
$ curl 'http://localhost:9090/api/v1/query_range?query=up&start=2015-07-01T20:10:30.781Z&end=2015-07-01T20:11:00.781Z&step=15s'
```

返回：
```json
{
   "status" : "success",
   "data" : {
      "resultType" : "matrix",
      "result" : [
         {
            "metric" : {
               "__name__" : "up",
               "job" : "prometheus",
               "instance" : "localhost:9090"
            },
            "values" : [
               [ 1435781430.781, "1" ],
               [ 1435781445.781, "1" ],
               [ 1435781460.781, "1" ]
            ]
         },
         {
            "metric" : {
               "__name__" : "up",
               "job" : "node",
               "instance" : "localhost:9091"
            },
            "values" : [
               [ 1435781430.781, "0" ],
               [ 1435781445.781, "0" ],
               [ 1435781460.781, "1" ]
            ]
         }
      ]
   }
}
```


本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)
