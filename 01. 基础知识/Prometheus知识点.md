# 简介

[Prometheus](https://links.jianshu.com/go?to=https%3A%2F%2Fprometheus.io) 是一套开源的系统监控报警框架。它始于2012年由 SoundCloud 创建，并作为社区开源项目进行开发。2016，Prometheus 正式加入 Cloud Native Computing Foundation (简称：CNCF)，成为受欢迎程度仅次于 Kubernetes 的项目。

作为新一代的监控系统，Prometheus 具有以下特点：

- 强大的多维数据库模型：
    1.时间序列数据通过 metric 名和【键/值】对来区分。
    2.所有的 metrics 都可以设置任意的多维标签。
    3.数据模型更随意，不需要刻意设置成以点分割的字符串。
    4.可以对数据模型进行聚合、切割、切片等操作。
    5.支持双精度浮点型，标签可以设为全 unicode。
- 灵活而强大的查询语言(PromQL，Query Language for Prometheus)：在同一个查询语句中，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。
- 易于管理：Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储，即单个 Prometheus server 节点是自治的。
- 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
- 使用 Pull(拉模式) 采集时间序列数据，可避免有问题的服务器向其推送坏的 metrics。
- 支持采用 push gateway 的方式将时间序列数据推送至 Prometheus server 端。
- 可通过服务发现和静态配置获取监控的 targets。
- 支持多种可视化图形界面。
- 易于伸缩。
- Federation 机制：允许一个 Prometheus server 获取另一个 Prometheus server 的 metrics。

需要指出的是，由于 Prometheus 采集数据可能会丢失，因此 Prometheus 不适用于对采集数据要求 100% 精确的场景，例如计费系统。但如果用于记录时间序列数据，Prometheus 是具有很大的查询优势。此外，Prometheus 适用于微服务框架。

# 组成与架构

Prometheus 生态圈中包含了多个组件，其中许多组件是可选的：

- Prometheus server：核心组件，用于抓取和存储时间序列数据。
- Client Libraries：客户端库，为需要监控的服务生成相应的 metrics 并暴露给 Prometheus server。当 Prometheus server 来 pull 时，直接返回实时状态的 metrics。
- Push gateway：主要用于短期的 jobs。由于这类 jobs 存活时间较短，可能在 Prometheus server 来 pull 之前就消失了。针对 push 系统设计， Short-lived jobs 可定时将 metrics push 到 Push gateway，再由 Prometheus server 从 Push gateway 上 pull metrics。这种方式主要用于服务层面的 metrics，对于机器层面的 metrics，需要使用 node exporter。
- Exporters：用于暴露已有的第三方服务的 metrics 给 Prometheus。
- Alertmanager：Prometheus 的报警组件，是与 Prometheus 组件相互分离的。Prometheus server 根据【告警规则】将 alerts 发送给 Alertmanager，Alertmanager 从 Prometheus server 接受到 alerts 后，进行去重、分组、降噪等处理，并将 alerts 通过路由发送到正确的接收器上，例如电子邮件、Slack、PaperDuty、HipChat、OpsGenie、WebHook 等。Alertmanager 还支持分组(Grouping)、抑制(Inhibition)、沉默(Silences)的机制。
- 一些其他工具的支持。

值得一提的是，大多数的 Prometheus 组件都是由 Go 语言编写的，这使得它们易于(作为二进制文件进行)构建和部署。



# 相关概念

Prometheus 的几个重要概念：Data model、Metric types、Jobs and instances。

## Data model

Prometheus 几乎将所有的数据存储为时间序列，每个时间序列都由 metrics 名和一组键值对(也称为标签)唯一标识的，不同的标签代表不同的时间序列。除了存储时间序列外，Prometheus 还可以根据查询结果生成临时派生的时间序列。

- Metric Names(metric 名)：该名字应该具有语义，一般用于表示 metric 的功能(例如：http_requests_total，表示 http 请求的总数)。其中，metric 名由 ASCII 字符、数字、下划线以及冒号组成，并且不能以数字开头，需要符合正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
- Labels(标签)：使同一个时间序列有了不同维护的识别。例如：http_requests_total{method="Get"}，表示所有 http 请求中的 Get 请求的总数。当 method="Post" 时，则是一个新的 metric。标签中的键由 ASCII 字符、数字以及下划线组成，并且不能以数字开头，需要符合正则表达式 [a-zA-Z_][a-zA-Z0-9_]*。
- Samples(样本)：Samples 构成实际的时间序列数据，每个时间序列数据包含一个 float64 值和一个毫秒级的时间戳。
- Notation(格式)：形如：<metric name>{<label name>=<label value>, ...}，示例：api_http_requests_total{method="POST", handler="/messages"}。

## Metric types

Prometheus 的 Client Libraries 提供了四种核心的 metric 类型：Counter、Gauge、Histogram、Summary。它们目前只在 Client Libraries 和 wire 协议中进行区分，Prometheus server 尚未使用类型信息，而是将所有的数据保存为无类型的时间序列。这个在将来或许会有所变化。

Prometheus 的 Client Libraries 提供的四种 metric 类型如下：

- Counter：一种累加的 metric，其值只能在重启时递增或重置为0。典型的应用如：请求的个数、结束的任务数、出现的错误数等。
   例如，查询 http_requests_total{method="get", job="Prometheus", handler="query"} 返回 8，10 秒后，再次查询，则返回 14。
- Gauge：一种常规的 metric，可以任意加减。典型的应用如：测量温度、当前内存的使用率；也用于“计数”，比如并发请求的数量。
   例如：go_goroutines{instance="172.17.0.2", job="Prometheus"} 返回值 147，10 秒后返回 124。
- Histogram：可理解为柱状图，提供 count 和 sum 全部值的功能，常用于跟踪事件发生的规模，典型应用如：请求耗时、响应大小。特别之处，可以对观察结果进行分组统计。
- Summary：与 Histogram 类似，提供 count 和 sum 全部值的功能，常用于跟踪事件发生的规模，典型应用如：请求耗时、响应大小。特别之处，提供百分位的功能，可按百分比划分跟踪的结果。

## Jobs and instances

instance：一个单独 scrape 的目标，一般对应于一个进程。
 job：一组同种类型的 instances(主要用于保证可扩展性和可靠性)。

当 scrape 目标时，Prometheus 会自动给这个 scrape 的时间序列附加一些标签(例如：instance、job)以便更好的分别。

## 示例说明

以实际的 metric 为例，对上述概念加以说明：

```bash
http_requests_total{code="2",handler="graph",instance="172.18.238.200:9090",job="prometheus"}
http_requests_total{code="2",handler="rules",instance="172.18.238.200:9090",job="prometheus"}
http_requests_total{code="2",handler="status",instance="172.18.238.200:9090",job="prometheus"}
```

可见，这三个 metric 的名字都一样，凭 handler 标签的值不同而被标识为不同的metrics。这类标签只会向上累加，属于 Counter 类型的 metric，且 metrics 中包含 instance 和 job 这两个标签。

