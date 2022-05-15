---
layout: post
title:  "VictoriaMetrics"
date:   2022-05-15 12:23:04 +0800
categories: linux
---

`VictoriaMetrics`: 一种快速、经济高效且可扩展的监控解决方案和时间序列数据库。

写入接口：
- Metrics scraping from Prometheus exporters.
- Prometheus remote write API.
- Prometheus exposition format.
- InfluxDB line protocol over HTTP, TCP and UDP.
- Graphite plaintext protocol with tags.
- OpenTSDB put message.
- HTTP OpenTSDB /api/put requests.
- JSON line format.
- Arbitrary CSV data.
- Native binary format

查询接口：
- Prometheus querying API
- Graphite API

【目录】
- [下载运行](#下载运行)
- [从 influx 代理处获取数据](#从-influx-代理处获取数据)
- [MetricsQL](#metricsql)
- [时间序列选择器](#时间序列选择器)
- [数学运算](#数学运算)
- [函数运算](#函数运算)
- [HTTP API](#http-api)
- [HTTP API 查询示例](#http-api-查询示例)
- [VMUI 数据查询与展示](#vmui-数据查询与展示)
- [参考资料](#参考资料)

## 下载运行

```bash
### -storageDataPath: 数据库存储路径
### -retentionPeriod: 数据保留周期
### -downsampling.period=30d:5m: 下采样，只有企业版提供
$ wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.77.0/victoria-metrics-arm64-v1.77.0.tar.gz
$ tar -zxf victoria-metrics-arm64-v1.77.0.tar.gz
$ ./victoria-metrics-prod \
  -storageDataPath=/home/yewq/victoriametrics/ \
  -retentionPeriod=1y
``` 

- [【文档】VictoriaMetrics: 如何启动](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-start-victoriametrics)
- [【文档】VictoriaMetrics: 保留周期](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#retention)
- [【软件】VictoriaMetrics Github 发布页面](https://github.com/VictoriaMetrics/VictoriaMetrics/releases)

## 从 influx 代理处获取数据

- 写入网址：http://10.0.1.102:8428/write
- 写入数据格式：line 格式

```
$ curl -X POST 'http://10.0.1.102:8428:8428/write' \
  -d 'foo,tag1=value1,tag2=value2 field1=12,field2=40'  
```

由于 influxdb 与 VictoriaMetrics 的时间序列格式不同，写入数据格式会被 VictoriaMetrics 自动进行转换：

```
### indluxdb 的 line 格式数据
foo,tag1=value1,tag2=value2 field1=12,field2=40
### 转换为 Prometheus 的 metrics 格式数据写入
foo_field1{tag1="value1", tag2="value2"} 12
foo_field2{tag1="value1", tag2="value2"} 40
### VictoriaMetrics 数据库中的存储格式
{"metric":{"__name__":"foo_field1","tag1":"value1","tag2":"value2"},"values":[12],"timestamps":[1560272508147]}
{"metric":{"__name__":"foo_field2","tag1":"value1","tag2":"value2"},"values":[40],"timestamps":[1560272508147]}
```

对于 Telegraf 代理插件，直接修改 Telegraf 配置文件，增加输出插件字段即可：

```ini
[[outputs.influxdb]]
  urls = ["http://10.0.1.102:8428"]
```

- [【文档】VictoriaMetrics: 如何从与 InfluxDB 兼容的代理（例如Telegraf ）发送数据](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#how-to-send-data-from-influxdb-compatible-agents-such-as-telegraf)
- [【手册】InfluxDB Line 格式](https://docs.influxdata.com/influxdb/v2.1/reference/syntax/line-protocol/)

## MetricsQL

`MetricsQL`: VictoriaMetrics 的查询语句格式。与 PromQL 类似，但提供了更多的函数运算，扩展了语法格式。由三部分组成：
1. 时间序列选择器
2. 数学运算
3. 函数运算

- [【文档】VictoriaMetrics MetricsQL](https://docs.victoriametrics.com/MetricsQL.html)
- [【博客】面向初学者的 PromQL 教程](https://valyala.medium.com/promql-tutorial-for-beginners-9ab455142085)

## 时间序列选择器

`Instant vector selectors`: 即时向量选择器。根据给定的时间戳（即时）选择包含单个样本值的时间序列，每个样本序列只一个时间戳+值。

```bash
### measurement: 测量名
### tag1: 标签名
### value1: 标签值字符串，可以是正则表达式
### [ @ <float_literal> ]: 指定 unix 时间戳，默认为当前系统时间。但返回的时间戳还是当前时间，但值是指定时间戳的值
### [ offset <duration> ]: 指定相对时间戳的偏移值
<measurement>{<tag1>="value1", <tag2>="value2" ...} [ @ <float_literal> ] [ offset <duration> ]
```

`Range Vector Selectors`: 范围向量选择器。根据给定的时间戳选择包含多个样本值的时间序列集合，每个样本序列集合包含多个时间戳+值。

```bash
### <instant_query>: 即时向量选择器
### <range>: 查询范围，如 5m 代表 5 分钟
### [<resolution>]: 查询分辨率
<instant_query> '[' <range> ':' [<resolution>] ']' [ @ <float_literal> ] [ offset <duration> ]
```

- [【文档】Prometheus 查询基础：即时矢量选择器](https://prometheus.io/docs/prometheus/latest/querying/basics/#instant-vector-selectors)
- [【文档】Prometheus 查询基础：范围矢量选择器](https://prometheus.io/docs/prometheus/latest/querying/basics/#range-vector-selectors)
- [【手册】UNIX 时间戳格式标准 5.8. Examples](https://www.ietf.org/rfc/rfc3339.txt)

## 数学运算

只有即时向量能进行数学运算，范围向量不能进行数学运算。  
运算规则：
1. 标量 & 即时向量：标量与即时向量中每个数据样本值进行运算。  
2. 即时向量 & 即时向量：根据标签进行一对一或一对多的向量匹配，匹配成功后进行数学运算。  

- [【文档】Prometheus 查询数学运算：向量匹配](https://prometheus.io/docs/prometheus/latest/querying/operators/#vector-matching)

## 函数运算

即时向量和范围向量均可以进行函数运算。不过不同的函数对输入向量格式有要求，详见函数手册。  
最常见的函数运算是速率函数 `rate(v range-vector)`，可以计算时间范围内平均每秒的增加速率，返回一个即时向量。

- [【文档】Prometheus 查询函数运算：rate() 函数](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate)

## HTTP API

使用 http 协议即可快速查询数据库，查询语句格式遵循 MetricsQL。

### `即时查询`：查询某一时刻的值。可以替代范围查询。

- `query`: MetricsQL。

```http
POST /api/v1/query HTTP/1.1
Host: 10.0.1.102:8428
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

query=cpu_usage_idle{host="kylin-gw102", cpu="cpu-total"}
```

### `范围查询`：查询某段时间内的值。可以被即时查询替代。

- `query`: MetricsQL。时间序列选择器只能使用即时向量选择器，范围向量选择器给定的时间范围会被覆盖。
- `start`: 起始时间戳
- `end`: 结束时间戳
- `step`: 查询步进

```http
POST /api/v1/query_range HTTP/1.1
Host: 10.0.1.102:8428
Content-Type: application/x-www-form-urlencoded
Content-Length: 133

query=cpu_usage_idle{cpu="cpu-total"}&start=2022-05-07T10:30:00+08:00&end=2022-05-07T10:31:00+08:00&step=1s
```

### `标签集查询`‘：查询一个测试名称下有哪些标签，方便后续根据标签名进一步查询值。

- `match[]`: 测量名

```
POST /api/v1/series HTTP/1.1
Host: 10.0.1.102:8428
Content-Type: application/x-www-form-urlencoded
Content-Length: 26

match[]=cpu_usage_idle
```

### `标签名称查询`：查询所有标签名，以便后续根据标签名进一步查询值。

```http
GET /api/v1/labels HTTP/1.1
Host: 10.0.1.102:8428
```

- [【文档】Prometheus 查询 HTTP API：即时查询](https://prometheus.io/docs/prometheus/latest/querying/api/#instant-queries)
- [【文档】Prometheus 查询 HTTP API：范围查询](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries)
- [【文档】Prometheus 查询 HTTP API：标签集查询](https://prometheus.io/docs/prometheus/latest/querying/api/#finding-series-by-label-matchers)
- [【文档】Prometheus 查询 HTTP API：标签名称查询](https://prometheus.io/docs/prometheus/latest/querying/api/#getting-label-names)

## HTTP API 查询示例

-  查询 cpu 的使用率（`即时查询`）：

```json
$ curl -X POST "http://10.0.1.102:8428/api/v1/query" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-binary 'query=(100 - cpu_usage_idle{host="kylin-gw102", cpu="cpu-total"})[1m:1s]'
{
    "status": "success",
    "data": {
        "resultType": "matrix",
        "result": [
            {
                "metric": {
                    "cpu": "cpu-total",
                    "db": "telegraf",
                    "host": "kylin-gw102"
                },
                "values": [
                    [
                        1652059345,
                        "0.18770530269999597"
                    ],
                    [
                        1652059346,
                        "0.5625"
                    ],
                    ......
                    ......
                    [
                        1652059405,
                        "0.4831670822999996"
                    ]
                ]
            }
        ]
    }
}
```

- 查询网络接口的传输速率（`即时查询`）：

```json
$ curl -X POST "http://10.0.1.102:8428/api/v1/query" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-binary 'query=rate(net_bytes_recv{host="kylin-gw102", interface="enp11s0f0"}[1s])[1m:1s]'
{
    "status": "success",
    "data": {
        "resultType": "matrix",
        "result": [
            {
                "metric": {
                    "db": "telegraf",
                    "host": "kylin-gw102",
                    "interface": "enp11s0f0"
                },
                "values": [
                    [
                        1652062185,
                        "1606"
                    ],
                    [
                        1652062186,
                        "8363"
                    ],
                    ......
                    ......
                    [
                        1652062245,
                        "1864"
                    ]
                ]
            }
        ]
    }
}
```

## VMUI 数据查询与展示

GRAPH 等价于 HTTP API 的 `范围查询`，JSON 等价于 HTTP API 的 `即时查询`。

- 网址：http://10.0.1.102:8428/vmui/
- Query 输入框：MetricsQL。时间序列选择器只能使用即时向量选择器，范围向量选择器给定的时间范围会被覆盖。
- Override step value: 查询步进

- [【文档】VictoriaMetrics vmui](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#vmui)

## 参考资料

1. [【主页】VictoriaMetrics](https://victoriametrics.com/)
2. [【文档】VictoriaMetrics](https://docs.victoriametrics.com/)
3. [【源码】VictoriaMetrics Github 仓库](https://github.com/VictoriaMetrics/VictoriaMetrics)
4. [【博客】浅析下开源时序数据库VictoriaMetrics的存储机制](https://zhuanlan.zhihu.com/p/368912946)