# SRS支持Prometheus Exporter

原创 ossrs SRS开源服务器 *2022-10-31 09:27* *发表于北京*

收录于合集



![图片](./prome/20221031144357.jpg)

> Written by 张俊勤, 孟晓伟.

SRS的可观测性是支撑业务的运营的能力，主要指监控(Prometheus Exporter)、分布式链路追踪(APM)、上下文日志(Cloud Logging)三个核心能力，以及基于这些能力的运营大盘、监控系统、问题排查、日志收集和分析等功能。

## Why Important?

可观测性，在云原生中有详细的定义，参考OpenTelemetry.io[1]，从可观测性上看其实就是三个问题：

- • 监控指标(Metrics)[2]：就是我们一般所理解的监控告警。监控一般是将一些数据聚合，体现系统在不同层面的状态，当达到某个状态后告警。比如区域的水位值，达到一定水位后就需要自动或人工扩容，或者调整调度降低这个区域的负载。

- • 分布式跟踪(Tracing)[3]：我们排查问题时，一般是按照会话或请求维度排查，在系统中会涉及多个服务器，比如播放一个流会经过API、调度、边缘、上游服务器、源站等，如何把这个全链路的信息给出来，就是分布式追踪(Tracing)。明显这是非常高效的解决问题的方法，问题迟迟得不到解决甚至不了了之，迟早会失去用户。

- • 日志(Logging)[4]: 就是我们一般所理解的日志，也是一般研发所依赖的排查问题的几乎唯一的方法。其实日志是最低效的方法，因为日志没有上下文，无法在分布式系统中分离出某个会话的多个日志。日志只有具备追踪的能力，或者在关联到Traceing中，这样才能更高效。

![图片](./prome/20221031144453.jpg)

> Note: 上图请参考Metrics, tracing, and logging[5]

> Note: SRS一直吹的可追踪日志，其实是在日志中加入了一定追踪能力，不过没有分布式追踪能力，也没有标准化（工具链就不完善，也无法让业务系统采用同样的日志，无法让整个系统具备可观测性）。

> Note: SRS的错误日志还有堆栈信息，这对于Java或Go等现代语言是很普通的能力，但是对于C++就是对问题排查非常有用的能力了。

回头看我们之前分析的几个问题，如何使用这三个能力来解决：

- • 收到了系统告警：根据告警的信息，比如是错误率上升，还是耗时增加，在追踪(Tracing)系统中过滤，可以看到具体的请求和会话，就可以获取相关的详细日志。

- • 用户反馈的具体问题：根据业务ID，查询到请求或会话的信息，在追踪(Tracing)系统中可以看到整个链路，也可以查询到每个环节的详细日志。

- • 需要重复开发运营系统问题：标准化的可观测，可以直接对接到Prometheus和Grafana，以及云存储比如CLS[6]，直接使用这些标准化的工具，不用再重复开发。

![图片](./prome/20221031144522.png)

> Note: 上图请参考Loki: Prometheus-inspired, open source logging for cloud natives[7]

> Note: 这套系统就没有问题吗？其实也有，那就是超级大规模的系统不能用，比如上千万并发的系统，产生的日志非常多，成本太高。如果你是SRS的用户，你就不用想了，你的系统达不到这么高量级。只有云计算本身的可观测性不能直接使用云原生的这套系统，而一般遵循这个标准定制。一般的业务规模，完全是够用的，而是成本非常非常低。

针对上面的问题，SRS的运营能力分成几个独立的部分，首先是提供了Prometheus可以对接的Exporter，Prometheus可以直接从SRS拉取监控数据，而不依赖外部第三方服务，如下图所示：

```
+-----+               +-----------+     +---------+
| SRS +--Exporter-->--| Promethus +-->--+ Grafana +
+-----+   (HTTP)      +-----------+     +---------+
```

> Note: Promethus是云原生的标准监控系统，在K8s中部署也可以使用这个能力，比如通过Pod发现和采集数据。

下面是详细的使用说明。

## Usage for SRS Exporter

首先，编译和启动SRS，要求`SRS 5.0.86+`：

```
./configure && make
env SRS_ENV_ONLY=on SRS_EXPORTER_ENABLED=on SRS_LISTEN=1935 \  
./objs/srs
```

> Note: 我们使用环境变量方式配置SRS，不依赖配置文件。当然使用`conf/prometheus.conf`启动也可以。

> Note: SRS启动成功后，可以打开http://localhost:9972/metrics验证，能看到返回指标数据就是成功了。

接着，我们启动FFmpeg推流：

```
docker run --rm -it ossrs/srs:encoder ffmpeg -re -i doc/source.flv -c copy \
  -f flv rtmp://host.docker.internal/live/livestream
```

然后，启动node_exporter[8]，收集节点的数据，这样和SRS的服务器数据可以形成完整的监控数据:

```
docker run --rm -p 9100:9100 prom/node-exporter
```

> Note: 用Docker启动node_exporter数据不准，需要特殊的权限而mac不支持。实际场景请使用二进制直接在主机上启动，可以从这里[9]下载对应系统的二进制。

> Note: node_exporter启动后，可以打开http://localhost:9100/metrics验证，能看到返回指标数据就是成功了。

最后，编写配置文件`prometheus.yml`，内容如下：

```
scrape_configs:
  - job_name: "node"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9100"]
  - job_name: "srs"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9972"]
```

> Note: 默认`scrape_interval`是1m即一分钟，为了测试方便我们设置为`5s`。

启动Prometheus：

```
docker run --rm -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 prom/prometheus
```

打开Prometheus: Targets[10]，可以看到抓取数据的状态。打开Prometheus: Graph[11]，输入一下语句，可以验证是否正常：

```
rate(srs_receive_bytes_total[10s])*8/1000
```

这个语句是计算输入的带宽，也就是直播流的码率，如下图所示：

![图片](./prome/20221031144809.png)

虽然Prometheus也能生成图，不过一般是使用Grafana对接Prometheus展示图表。

## Usage for Grafana

首先启动Grafana：

```
docker run --rm -it -p 3000:3000 \
  -e GF_SECURITY_ADMIN_USER=admin \
  -e GF_SECURITY_ADMIN_PASSWORD=12345678 \
  -e GF_USERS_DEFAULT_THEME=light \
  grafana/grafana
```

然后打开Grafana页面：http://localhost:3000/

输入用户名`admin`，以及密码`12345678`就可以进入Grafana后台了。

执行命令添加[12]Prometheus的DataSource：

```
curl -s -H "Content-Type: application/json" \
    -XPOST http://admin:12345678@localhost:3000/api/datasources \
    -d '{
    "name": "prometheus",
    "type": "prometheus",
    "access": "proxy", "isDefault": true,
    "url": "http://host.docker.internal:9090"
}'
```

执行命令导入[13]HelloWorld图表：

```
curl https://raw.githubusercontent.com/ossrs/srs-grafana/main/dashboards/helloworld.json | \
curl -s -H "Content-Type: application/json" \
    -XPOST http://admin:12345678@localhost:3000/api/dashboards/db \
    --data-binary @-
```

导入后就可以在仪表盘中[14]看到了，如下图所示：

![图片](./prome/20221031144909.png)

我们还提供了更加完整的仪表盘，可以在srs-grafana[15]中看到，如下图所示：

![图片](./prome/20221031144928.png)

欢迎一起来完善SRS仪表盘。

#### 引用链接

`[1]` OpenTelemetry.io: *https://opentelemetry.io*  
`[2]` 监控指标(Metrics): *https://opentelemetry.io/docs/concepts/observability-primer/#reliability--metrics*  
`[3]` 分布式跟踪(Tracing): *https://opentelemetry.io/docs/concepts/observability-primer/#distributed-traces*  
`[4]` 日志(Logging): *https://opentelemetry.io/docs/concepts/observability-primer/#logs*  
`[5]` Metrics, tracing, and logging: *https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html*  
`[6]` CLS: *https://cloud.tencent.com/product/cls*  
`[7]` Loki: Prometheus-inspired, open source logging for cloud natives: *https://grafana.com/blog/2018/12/12/loki-prometheus-inspired-open-source-logging-for-cloud-natives/*  
`[8]` node_exporter: *https://github.com/prometheus/node_exporter*  
`[9]` 这里: *https://github.com/prometheus/node_exporter/releases*  
`[10]` Prometheus: Targets: *http://localhost:9090/targets*  
`[11]` Prometheus: Graph: *http://localhost:9090/graph*  
`[12]` 添加: *https://grafana.com/docs/grafana/latest/developers/http_api/data_source/#create-a-data-source*  
`[13]` 导入: *https://grafana.com/docs/grafana/latest/developers/http_api/dashboard/#create--update-dashboard*  
`[14]` 仪表盘中: *http://localhost:3000/dashboards*  
`[15]` srs-grafana: *https://github.com/ossrs/srs-grafana/tree/main/dashboards*
