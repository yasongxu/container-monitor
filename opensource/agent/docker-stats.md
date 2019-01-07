## 前言

传统虚机监控一般采用类似Zabbix的方案，但容器出现之后，再使用Zabbix agent来采集数据的话就显得有些吃力了，如果每个容器都像OS那样监控，则metric数量将会非常巨大，而且这些数据很可能几分钟之后就没有意义了（容器已经停止或漂移），且容器的指标汇总更应该是按照APP甚至POD维度。

如果只是过渡方案，或者想将容器监控统一到公司现有的Zabbix中，可以参考[zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring)，有很多模板如：[zabbix-template-app-docker.xml](https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker.xml)

参考文章：https://segmentfault.com/a/1190000007568413

## Docker原生监控

常用方式：

* docker ps/top/logs
* docker stats
* docker Remote API
* docker 伪文件系统

**docker stats**

该命令默认以流式方式输出，如果想打印出最新的数据并立即退出，可以使用 no-stream=true 参数。

可以指定一个已停止的容器，但是停止的容器不返回任何数据。

例如：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/2bafa9c5995f2a78a0273ec3266b9f2c.png)

**Remote API**

Docker Remote API是一个取代远程命令行界面（rcli）的REST API
如：`curl http://127.0.0.1:4243/containers/json`

可以使用API来获取监控数据并集成到其他系统，注意不要给Docker daemon带来性能负担，如果你一台主机有很多容器，非常频繁的采集可能会大量占据CPU

**伪文件系统**

`以下操作的环境为：Centos7系统 docker17.03版本`

docker stats的数据来自于/sys/fs/cgroup下的文件

* mem usage那一列的值，来自于
`/sys/fs/cgroup/memory/docker/[containerId]/memory.usage_in_bytes`

* 如果没限制内存，Limit = machine_mem，否则来自于
`/sys/fs/cgroup/memory/docker/[id]/memory.limit_in_bytes` 

* 内存使用率 = memory.usage_in_bytes/memory.limit_in_bytes

一般情况下，cgroup文件夹下的内容包括CPU、内存、磁盘、网络等信息：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/f5f36198ff40fd2aac2dd2a6a3656997.png)

![](http://www.xuyasong.com/wp-content/uploads/2019/01/f6d0df08c95c331470b01ac9e7b6af17.png)

如memory下的文件有：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/56f83bece0395d763e1178453b47196f.png)

几个常用的指标含义：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/6462600e967476ecdf1188eececafc49.png)

memory.stat中的信息是最全的：

![](http://www.xuyasong.com/wp-content/uploads/2019/01/a5b7fe3eae7680a3d5f38ca798ecf905.png)


更多资料参考：[cgroup memory](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory)

原理分析：[Libcontainer 深度解析](https://www.infoq.cn/article/docker-container-management-libcontainer-depth-analysis)

## 总结

优缺点：

* 优点：原生，很方便的看到当前宿主机上所有容器的CPU、内存、网络流量等数据。
* 缺点：只能统计当前宿主机的所有容器，数据是实时的，没有存储，没有报警，没有可视化。

备注：

* 1.如果你没有限制容器内存，那么docker stats将显示您的主机的内存总量。但它并不意味着你的每个容器都能访问那么多的内存
* 2.默认时stats命令会每隔1秒钟刷新一次，如果只看当前状态：docker stats --no-stream
* 3.指定查看某个容器的资源可以指定名称或PID: docker stats --no-stream registry 1493
