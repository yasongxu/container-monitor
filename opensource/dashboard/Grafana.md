## 概述

[Grafana](https://grafana.com/) 是一个开源的，可以用于大规模指标数据的可视化项目，甚至还能对指标进行报警。基于友好的 Apache License 2.0 开源协议，目前是prometheus监控展示的首选。优点如下：

##### 1.使用：

* 配置方便：支持Dashboard、Panel、Row等组合，且支持折线图、柱状图等多种图例
* 图表漂亮：可以选择暗黑系或纯白系，你也可以自己定义颜色
* 模板很多：grafana模板很活跃，有很多[用户贡献的面板](https://grafana.com/dashboards)，直接导入就能用
* 支持多种数据源：grafana作为展示面板，支持很多数据源，如Graphite、ES、Prometheus等
* 权限管理简单：有admin、viewer等多种角色管控

##### 2.二次开发：

如果默认的grafana不能满足你的需求，要二次开发，官方也提供了很多支持：

* 协议为Apache License 2.0：商业友好，随便改吧，改完拿去卖也行。
* 完善的API调用：权限、面板、用户、报警都支持api调用。
* 多种鉴权方式：OAuth、LADP、Proxy多种方式，你可以接入自己公司的鉴权系统
* 插件开发：如果你不想直接改代码，可以做自己的插件
* go+Angular+react：常用的技术栈，方便二次开发

prometheus + grafana 做为监控组合很方便，很强大，改造了鉴权之后更加香。一开始我们还尝试使用grafana自带的报警功能，可惜比较鸡肋，无法用于生产，报警的issue一大堆官方也不想修改，作罢

## 部署

**步骤一：安装grafana**

Grafana提供了很多种[部署方式](https://grafana.com/docs/installation/docker/)，如果你的展示报警是在K8S集群外，可以二进制直接部署，如果grafana本身在集群内，或者管理端也是k8s集群，可以用yaml部署：

Deployment配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: kube-system
  name: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      namespace: kube-system
      annotations:
        grafana-version: '1.0'
      name: grafana
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:5.1.0
        imagePullPolicy: Always
        securityContext:
          runAsUser: 0
        env: 
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: "admin"
        ports:
        - name: grafana
          containerPort: 3000
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "2048Mi"
            cpu: "1024m"
```

Service配置：

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: grafana
  annotations:
    prometheus.io/scrape: 'true'
  labels:
    name: grafana
spec:
  selector:
    app: grafana
  type: LoadBalancer
  ports:
  - name: grafana
    protocol: TCP
    port: 3000
```

grafana配置文件的字段含义：


```text
app_mode：应用名称，默认是production
 
[path]
data：一个grafana用来存储sqlite3、临时文件、回话的地址路径
logs：grafana存储logs的路径
 
[server]
http_addr：监听的ip地址，，默认是0.0.0.0
http_port：监听的端口，默认是3000
protocol：http或者https，，默认是http
domain：这个设置是root_url的一部分，当你通过浏览器访问grafana时的公开的domian名称，默认是localhost
enforce_domain：如果主机的header不匹配domian，则跳转到一个正确的domain上，默认是false
root_url：这是一个web上访问grafana的全路径url，默认是%(protocol)s://%(domain)s:%(http_port)s/
router_logging：是否记录web请求日志，默认是false
cert_file：如果使用https则需要设置
cert_key：如果使用https则需要设置
 
[database]
grafana默认需要使用数据库存储用户和dashboard信息，默认使用sqlite3来存储，你也可以换成其他数据库
type：可以是mysql、postgres、sqlite3，默认是sqlite3
path：只是sqlite3需要，定义sqlite3的存储路径
host：只是mysql、postgres需要，默认是127.0.0.1:3306
name：grafana的数据库名称，默认是grafana
user：连接数据库的用户
password：数据库用户的密码
ssl_mode：只是postgres使用
 
 
[security]
admin_user：grafana默认的admin用户，默认是admin
admin_password：grafana admin的默认密码，默认是admin
login_remember_days：多少天内保持登录状态
secret_key：保持登录状态的签名
disable_gravatar：
 
 
[users]
allow_sign_up：是否允许普通用户登录，如果设置为false，则禁止用户登录，默认是true，则admin可以创建用户，并登录grafana
allow_org_create：如果设置为false，则禁止用户创建新组织，默认是true
auto_assign_org：当设置为true的时候，会自动的把新增用户增加到id为1的组织中，当设置为false的时候，新建用户的时候会新增一个组织
auto_assign_org_role：新建用户附加的规则，默认是Viewer，还可以是Admin、Editor
 
 
[auth.anonymous]
enabled：设置为true，则开启允许匿名访问，默认是false
org_name：为匿名用户设置组织名称
org_role：为匿名用户设置的访问规则，默认是Viewer
 
 
[auth.github]
针对github项目的，很明显，呵呵
enabled = false
allow_sign_up = false
client_id = some_id
client_secret = some_secret
scopes = user:email
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
team_ids =
allowed_domains =
allowed_organizations =
 
 
[auth.google]
针对google app的，呵呵
enabled = false
allow_sign_up = false
client_id = some_client_id
client_secret = some_client_secret
scopes = https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email
auth_url = https://accounts.google.com/o/oauth2/auth
token_url = https://accounts.google.com/o/oauth2/token
api_url = https://www.googleapis.com/oauth2/v1/userinfo
allowed_domains =
 
 
[auth.basic]
enabled：当设置为true，则http api开启基本认证
 
 
[auth.ldap]
enabled：设置为true则开启LDAP认证，默认是false
config_file：如果开启LDAP，指定LDAP的配置文件/etc/grafana/ldap.toml
 
 
[auth.proxy]
允许你在一个HTTP反向代理上进行认证设置
enabled：默认是false
header_name：默认是X-WEBAUTH-USER
header_property：默认是个名称username
auto_sign_up：默认是true。开启自动注册，如果用户在grafana DB中不存在
 
[analytics]
reporting_enabled：如果设置为true，则会发送匿名使用分析到stats.grafana.org，主要用于跟踪允许实例、版本、dashboard、错误统计。默认是true
google_analytics_ua_id：使用GA进行分析，填写你的GA ID即可
 
 
[dashboards.json]
如果你有一个系统自动产生json格式的dashboard，则可以开启这个特性试试
enabled：默认是false
path：一个全路径用来包含你的json dashboard，默认是/var/lib/grafana/dashboards
 
 
[session]
provider：默认是file，值还可以是memory、mysql、postgres
provider_config：这个值的配置由provider的设置来确定，如果provider是file，则是data/xxxx路径类型，如果provider是mysql，则是user:password@tcp(127.0.0.1:3306)/database_name，如果provider是postgres，则是user=a password=b host=localhost port=5432 dbname=c sslmode=disable
cookie_name：grafana的cookie名称
cookie_secure：如果设置为true，则grafana依赖https，默认是false
session_life_time：session过期时间，默认是86400秒，24小时
 
 
以下是官方文档没有，配置文件中有的
[smtp]
enabled = false
host = localhost:25
user =
password =
cert_file =
key_file =
skip_verify = false
from_address = admin@grafana.localhost
 
[emails]
welcome_email_on_sign_up = false
templates_pattern = emails/*.html
 
 
[log]
mode：可以是console、file，默认是console、file，也可以设置多个，用逗号隔开
buffer_len：channel的buffer长度，默认是10000
level：可以是"Trace", "Debug", "Info", "Warn", "Error", "Critical"，默认是info
 
[log.console]
level：设置级别
 
[log.file]
level：设置级别
log_rotate：是否开启自动轮转
max_lines：单个日志文件的最大行数，默认是1000000
max_lines_shift：单个日志文件的最大大小，默认是28，表示256MB
daily_rotate：每天是否进行日志轮转，默认是true
max_days：日志过期时间，默认是7,7天后删除

```


注意的几个点：

1. 对所有的资源都做request、limit限制，防止耗尽主机资源
2. grafana的一些配置可以通过[变量](https://grafana.com/docs/installation/configuration/)传入：如admin的密码GF_SECURITY_ADMIN_PASSWORD
3. 如果要对grafana的数据进行持久化，建议挂volume或者外部存储，持久化的内容一般都是面板配置。监控面板的配置可以导入导出
4. securityContext：因为版本问题，如果提示grafana的权限不足，可以配置runAsUser: 0


创建了grafana之后，可以通过service暴露的端口地址查看页面:

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/9004c940-6847-4c6f-8a96-b8656b90f706.jpg)

登录成功后，会显示需要初始化的内容

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/65a45865-1239-4a4c-ad0d-a0c736504ad1.jpg)


**步骤二：配置数据源**


grafana支持多种数据源，可以在“type”的下拉框选项中看到，这里我们选择prometheus作为数据源。HTTP的访问方式选择proxy，URL填写grafana能访问到的地址。

```因为grafana和prometheus都在同一个k8s集群中，这里用svc地址```


![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/b93da754-9100-4cb1-b692-2d1c00158aaf.jpg)


点击“save and test”测试连接性。


**步骤三：创建面板**


![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/9c27103e-ca3f-49ed-9dbf-188d3d5c53b3.jpg)


有了数据源，接下来就是如何更好地展示数据，grafana支持多种类型的图表，如Graph、singlestat、Table等。可以组合出多种形式。这里先创建一个Demo，保存现有模板的快捷键是Ctrl + S

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/50beb30e-6111-4053-a1fb-a756c2b0414d.jpg)


你的所有面板都可以在左上角的下拉框中找到：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/eae7ce0b-4d0c-482b-a70c-a8527a5bd3fb.jpg)


我们还可以导入现有的面板（Dashboard），大家贡献的模板地址：https://grafana.com/dashboards

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/24f2777b-10fc-48ab-9847-76c4a4f2667c.jpg)

选择左上角的➕号，然后import，有两种形式的导入：

* URL直接导入：粘贴对应模板的url，前提是grafana必须能访问外网。
* 上传json文件：一般是迁移时使用，把现有的json导出，再导入


grafana的面板、图表有很多配置，接下来我们说几个常用的配置项


## 常用配置

示例：K8S的节点数量趋势图

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/3c30ffc8-de9b-433b-8aa9-d100f2f2f1cd.jpg)

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/5bde561a-cf70-4037-8321-714fdba9f244.jpg)


编辑图表时默认进入的是Metric，内容包括：

1. Data Source：填写刚刚配置过的数据源的名称，这里是prometheus
2. A和B指的是这张图有几条线，这里是A一条线、右侧的小眼睛代表是否隐藏该线
3. A线中，输入框填写的是查询语句，这里指合法的promsql，legend format指A线的显示名称

除了Metric之外，还有几个选项，含义分别是：

* General：一般是将其他项都配置好后再回头设置General，而在General配置标签中一般只会用到“Title”，就是图表的标题，如这里的Node数量

``` text
Title：图表标题；
Description：图表描述；
Span：图表间隔，无需设定，在前端可手动调整图表大小；
Height：图表高度，无需设定，在前端可手动调整图表高度；
Transparent：背景是否透明，默认情况下不勾选，如果觉得不需要深灰色背景，可以勾选此项；
Templating和Drilldown / detail link用处不大，忽略即可。
```

* Metric：比较重要，配指标表达式和指标线的地方，上边已经举例
* Axes：配置数据轴的地方，如偏移缩放，格式转换
* Legend：图例，是否限制以及显示的方式
* Display: 展示相关的配置，如线条宽度，排序方式、空值处理
* Alert: 报警配置，grafana算是少有的展示图表支持报警的，但他的报警只支持到单图表，无法嵌套模板，有点鸡肋。如左上角有筛选node的下拉框，图表又传入了变量时，如果配置报警，是配置失败的。报错为：“Template variables are not supported in alert queries”
* TimeRange: 配置单图表的展示时间，如24h内的数据

```
Override relative time:  覆盖右上角选择的时间，设置要显示的时间范围，这里我设置24h（单位自己可选）。 
Add time shift:  这里是偏移量设置，比如填写2h表示不显示最近2h的数据。 
Hide time ocerride info:  上边相对时间设置之后在graph中会显示本图表的时间信息，在此处选择后可以把显示的信息隐藏掉

```

更多详细的配置可以查看：[https://grafana.com/docs/features/panels/graph/](https://grafana.com/docs/features/panels/graph/)

变量配置：

对于一些复杂场景，可能需要传入变量，如有多台机器，每台机器都要展示其cpu内存等指标。而机器列表又是动态的，这个时候就可以使用变量，示例：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/432ac5d4-64bb-493b-bb7c-ebb00d98507e.jpg)


首先在该面板的setting中选择variables，注意是该面板的设置，不是全局设置

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/a52a7a28-b6dc-4c1f-8851-46da286483a1.jpg)

填写名称，下拉框选项的数据获取表达式，刷新周期，是否有ALL选项等，然后保存

接下来在具体的图表中使用该变量

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/2afe913a-3ada-4b1a-a756-65110099a1f4.jpg)


在metric中，将变量$Node写在表达式中做匹配即可。

grafana的变量支持高级匹配，如$Node.*代表以Node开头的字符，利用变量的方式，可以实现多级筛选，满足更复杂的需求，如pod资源的查看

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/a60551dd-4aca-4c28-a50f-7dde11b0b9da.jpg)


关于变量的更多信息：[what-is-a-variable](https://grafana.com/docs/reference/templating/#what-is-a-variable)

## 报警配置

grafana在v4.0版本开始引入了报警功能。

还是以Node节点数为例，我们配置一条规则：


```
当可用节点数小于3时，报警给demo@126.com
```

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/0d753bfa-869d-40d2-b516-44ee8fabed69.jpg)


alert支持avg、sum等表达式，不过持续时间依赖数据本身的采集频率。需要多测试一下。

Notifications：配置报警的收件组和详细内容。而报警收件人的配置在专门的Alerting页面上

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/15808c2e-61ea-441a-b7a8-6abf48fe4a5e.jpg)


Alert Rules：已经配置的报警规则，并展示其触发状态。

报警邮件的样式：

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/d01935f8-9b7f-4ca1-9147-674e6cae2a1a.jpg)


**模板变量报警**

以上的报警配置方式只适合没有变量传入的图表，如果遇到上边提到的选择node，传入变量的图表，就没办法支持了。

![](http://vermouth-blog-image.oss-cn-hongkong.aliyuncs.com/monitor/2f4b7d3e-a798-4ee7-9cd7-548dbddb453a.jpg)

相关issue：https://github.com/grafana/grafana/issues/9334

官方对这个功能解释了一堆，最新版本仍然没有支持。借用issue的一句话哈哈哈

```
It would be grate to add templates support in alerts. Otherwise the feature looks useless a bit.

```


本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)

