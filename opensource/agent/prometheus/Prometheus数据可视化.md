
### 一. 概述
Prometheus自带了一个web服务，包括一个默认的dashboard，可以使用表达式查询并进行图表可视化，默认服务的地址为：http://prometheus_ip:9090

如下图：

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15516976087780.jpg)

自带的web展示一般只用于表达式快速输入或者临时调试，因为默认服务没有鉴权，且图表表达能力有限，因此不会作为线上可视化方案，正式的监控数据可视化一般使用Grafana来配套

prometheus可视化方案：

* 自带web服务：在验证指标时是非常好用的，grafana虽然是作为可视化展示，但一般是先确认表达式，才去配置到grafana面板
* grafana可视化
* [Console templates](https://prometheus.io/docs/visualization/consoles/):官方给的一种选择，使用go templete来实现，使用难度较大，不太推荐
* [promviz](https://github.com/nghialv/promviz)：开源项目，不算是监控图，可以做集群实时流量的可视化。

### 二. Grafana可视化

[Grafana](https://grafana.com/) 是一个开源的图表可视化系统，简单说图表配置比较方便、生成的图表比较漂亮。并且模板众多，默认支持了prometheus作为数据源，也是prometheus官方推荐方案

这里只对grafana做简单介绍，更多详细的内容参考[展示-Grafana](txxxx)

##### 2.1 部署

grafana是很成熟的(商业)项目，可以在官网[下载客户端](https://grafana.com/get)，或者在[github主页](https://github.com/grafana/grafana)自己build为镜像。

主要的配置文件为conf文件夹下的[defaults.ini文件](https://github.com/grafana/grafana/blob/master/conf/defaults.ini)，常用的配置可以配置在文件中，如果是docker运行或者在k8s中运行，可以使用[env的方式](http://docs.grafana.org/installation/configuration/)，传入全局变量，将覆盖原有的defaults.ini配置。

使用docker运行：

```bash
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

访问：http://127.0.0.1:3000

账号：admin
密码：admin

##### 2.2 配置

**第一步：配置数据源**

进入grafana后，第一步需要配置数据源，grafana默认支持prometheus作为数据源，因此Type直接选择Prometheus

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517017792219.jpg)


Http的url地址为prometheus的服务地址，如果是同一个pod内，可以127.0.0.1:9090，不同pod的话，可以使用svc地址：http://prometheus.kube-system.svc.cluster.local:9090

数据源配置后，点击save&test，可以验证数据源是否可用：

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517019316531.jpg)


**第二步：配置面板：**

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517020165486.jpg)


点击左侧的加号，可以添加面板(dashboard)，并在该面板中添加各种类型的图表。

对于面板，可以设置变量，用于下拉框筛选等场景，如设置机器变量：节点信息


![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517027323228.jpg)


然后使用该变量，配置查询语句：得到各节点的cpu使用率

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517027870215.jpg)



##### 面板demo

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517028260767.jpg) 

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15518793945999.jpg)


配置完成后，可以出现类似图表，可以点击分享按钮，将本面板分享为json文件

![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517028746898.jpg)


也可以筛选时间周期，设置自动刷新


![](http://www.xuyasong.com/wp-content/uploads/2019/03/15517028992578.jpg)


上图的json文件如下，仅供参考(需要安装node-exporter)

json文件：https://raw.githubusercontent.com/yasongxu/container-monitor/master/data/grafana-demo.json


本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)
