
## 概述


分布式追踪是一种跟踪应用程序请求流经不同服务（如前端、后端、数据库等）的过程。它是一个强大的工具，可以帮助您了解应用程序的工作原理并调试性能问题。


Quickwit 是一个用于索引和搜索非结构化数据的云原生引擎，这使其非常适合用作追踪数据的后端。


此外，Quickwit 本地支持 [OpenTelemetry gRPC 和 HTTP（仅 protobuf）协议](https://github.com) 以及 [Jaeger gRPC API（仅 SpanReader）](https://github.com)。**这意味着您可以使用 Quickwit 存储追踪数据，并通过 Jaeger UI 查询这些数据**。


* [https://opentelemetry.io/docs/reference/specification/protocol/otlp/](https://github.com)
* [https://www.jaegertracing.io/](https://github.com)


[![image](https://img2024.cnblogs.com/blog/436453/202408/436453-20240830134344582-483393649.webp)](https://github.com)


### 将 Quickwit 连接到 Jaeger


Quickwit 实现了一个与 Jaeger UI 兼容的 gRPC 服务。您只需要将 Jaeger 配置为使用 `grpc-plugin` 类型的（跨度）存储，就能够查看存储在任何匹配模式 `otel-traces-v0_*` 的 Quickwit 索引中的追踪数据。


官方制作了一个关于 [如何将 Quickwit 连接到 Jaeger UI](https://github.com) 的教程，将引导您完成整个过程。


* [https://quickwit.io/docs/distributed\-tracing/plug\-quickwit\-to\-jaeger](https://github.com)


### 向 Quickwit 发送追踪数据


* [使用 OTEL collector](https://github.com)
	+ [https://quickwit.io/docs/distributed\-tracing/send\-traces/using\-otel\-collector](https://github.com)
* [使用 Python OTEL SDK](https://github.com)
	+ [https://quickwit.io/docs/distributed\-tracing/send\-traces/using\-otel\-sdk\-python](https://github.com)


## 将 Quickwit 连接至 Jaeger


我们将 Quickwit 的追踪数据发送到 Jaeger 并进行分析，这将生成新的追踪数据以供分析 😃


### 启动 Quickwit


首先，启动一个启用了 OTLP 服务的 [Quickwit 实例](https://github.com)：


* [https://quickwit.io/docs/get\-started/installation](https://github.com)



```
QW_ENABLE_OPENTELEMETRY_OTLP_EXPORTER=true \
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:7281 \
./quickwit run

```

我们还设置了 `QW_ENABLE_OPENTELEMETRY_OTLP_EXPORTER` 和 `OTEL_EXPORTER_OTLP_ENDPOINT` 环境变量，以便 Quickwit 将其自身的追踪数据发送给自己。


### 启动 Jaeger UI


让我们使用 Docker 启动一个 Jaeger UI 实例。在这里我们需要告知 Jaeger 使用 Quickwit 作为其后端。


由于一些与容器网络相关的特殊性，我们需要在 MacOS 和 Windows 上采用一种方法，在 Linux 上采用另一种方法。


#### MacOS 与 Windows


我们可以依赖 `host.docker.internal` 来获取指向我们 Quickwit 服务器的 Docker 桥接 IP 地址。



```
docker run --rm --name jaeger-qw \
    -e SPAN_STORAGE_TYPE=grpc-plugin \
    -e GRPC_STORAGE_SERVER=host.docker.internal:7281 \
    -p 16686:16686 \
    jaegertracing/jaeger-query:latest

```

#### Linux


默认情况下，Quickwit 监听 `127.0.0.1`，并且不会响应指向 Docker 桥接 (`172.17.0.1`) 的请求。解决此问题有多种方法。最简单的方法可能是使用主机网络模式。



```
docker run --rm --name jaeger-qw  --network=host \
    -e SPAN_STORAGE_TYPE=grpc-plugin \
    -e GRPC_STORAGE_SERVER=127.0.0.1:7281 \
    -p 16686:16686 \
    jaegertracing/jaeger-query:latest


```

### 在 Jaeger UI 中搜索追踪数据


由于 Quickwit 会索引其自身的追踪数据，因此在大约 5 秒后（这是 Quickwit 完成首次提交所需的时间），您应该能够在 Jaeger UI 中看到这些数据。


打开位于 [http://localhost:16686](https://github.com) 的 Jaeger UI 并搜索追踪数据！通过执行搜索查询，您将看到 Quickwit 自身的追踪数据：


* `find_traces` 是在 Jaeger UI 中搜索追踪数据时调用的端点，随后它会调用 `find_trace_ids`。
* `find_traces_ids` 对跨度执行聚合查询以获取唯一的追踪 ID。
* `root_search` 是 Quickwit 的搜索入口点。它会在每个分片（索引的一部分）上并行地或分布式地调用搜索，如果只有一个节点，则仅在本地调用。
* `leaf_search` 是每个节点上的搜索入口点。它会在每个分片上调用 `leaf_search_single_split`。
* `leaf_search_single_split` 是分片上的搜索入口点。它会依次调用 `warmup` 和 `tantivy_search`。
* `warmup` 是搜索的预热阶段。它预取执行搜索查询所需的数据。
* `tantivy_search` 是搜索的执行阶段。它使用 [Tantivy](https://github.com) 高速执行搜索查询。


[![image](https://img2024.cnblogs.com/blog/436453/202408/436453-20240830134403904-344974851.webp)](https://github.com)


## 使用 OTEL Collector


如果您已经有自己的 OpenTelemetry Collector 并希望将跟踪数据导出到 Quickwit，您需要在 `config.yaml` 中添加一个新的 OTLP gRPC 导出器：


**macOS/Windows**



```
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:

exporters:
  otlp/quickwit:
    endpoint: host.docker.internal:7281
    tls:
      insecure: true
    # By default, traces are sent to the otel-traces-v0_7.
    # You can customize the index ID By setting this header.
    # headers:
    #   qw-otel-traces-index: otel-traces-v0_7

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/quickwit]

```

**Linux**



```
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:

exporters:
  otlp/quickwit:
    endpoint: 127.0.0.1:7281
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/quickwit]

```

### 测试您的 OTEL 配置


1. [安装](https://github.com) 并启动一个 Quickwit 服务器：
	* [https://quickwit.io/docs/get\-started/installation](https://github.com)



```
./quickwit run

```

2. 使用之前的配置启动一个收集器：


**macOS/Windows**



```
docker run -v ${PWD}/otel-collector-config.yaml:/etc/otelcol/config.yaml -p 4317:4317 -p 4318:4318 -p 7281:7281 otel/opentelemetry-collector

```

**Linux**



```
docker run -v ${PWD}/otel-collector-config.yaml:/etc/otelcol/config.yaml --network=host -p 4317:4317 -p 4318:4318 -p 7281:7281 otel/opentelemetry-collector

```

3. 使用 cURL 向收集器发送一个跟踪：



```
curl -XPOST "http://localhost:4318/v1/traces" -H "Content-Type: application/json" \
--data-binary @- << EOF
{
 "resource_spans": [
   {
     "resource": {
       "attributes": [
         {
           "key": "service.name",
           "value": {
             "string_value": "test-with-curl"
           }
         }
       ]
     },
     "scope_spans": [
       {
         "scope": {
           "name": "manual-test"
         },
         "spans": [
           {
             "time_unix_nano": "1678974011000000000",
             "observed_time_unix_nano": "1678974011000000000",
             "start_time_unix_nano": "1678974011000000000",
             "end_time_unix_nano": "1678974021000000000",
             "trace_id": "3c191d03fa8be0653c191d03fa8be065",
             "span_id": "3c191d03fa8be065",
             "kind": 2,
             "events": [],
             "status": {
               "code": 1
             }
           }
         ]
       }
     ]
   }
 ]
}
EOF

```

您应该会在 Quickwit 服务器上看到类似于以下的日志：



```
2023-03-16T13:44:09.369Z  INFO quickwit_indexing::actors::indexer: new-split split_id="01GVNAKT5TQW0T2QGA245XCMTJ" partition_id=6444214793425557444

```

这意味着 Quickwit 已经接收到跟踪并创建了一个新的分片。在搜索跟踪之前，请等待分片发布。


## 使用 OTEL SDK \- Python


在本教程中，我们将向您展示如何使用 OpenTelemetry 为 Python [Flask](https://github.com) 应用程序进行仪器化，并将跟踪数据发送到 Quickwit。本教程受到 [Python OpenTelemetry](https://github.com) 文档的启发，感谢 OpenTelemetry 团队！


* [https://flask.palletsprojects.com/en/2\.2\.x/](https://github.com)
* [https://opentelemetry.io/docs/instrumentation/python/getting\-started/](https://github.com)


### 前提条件


* 已安装 Python3
* 已安装 Docker


### 启动一个 Quickwit 实例


[安装 Quickwit](https://github.com) 并启动一个 Quickwit 实例：


* [https://quickwit.io/docs/main\-branch/get\-started/installation](https://github.com)



```
./quickwit run

```

### 启动 Jaeger UI


让我们使用 Docker 启动一个 Jaeger UI 实例。在这里我们需要告知 Jaeger 使用 Quickwit 作为其后端。


由于容器网络的一些特殊性，我们将在 MacOS 和 Windows 以及 Linux 上采用不同的方法。


#### MacOS 和 Windows


我们可以依赖 `host.docker.internal` 获取指向我们 Quickwit 服务器的 Docker 桥接 IP 地址。



```
docker run --rm --name jaeger-qw \
    -e SPAN_STORAGE_TYPE=grpc-plugin \
    -e GRPC_STORAGE_SERVER=host.docker.internal:7281 \
    -p 16686:16686 \
    jaegertracing/jaeger-query:latest

```

#### Linux


默认情况下，Quickwit 监听的是 `127.0.0.1`，并且不会响应指向 Docker 桥接 (`172.17.0.1`) 的请求。有多种方法可以解决这个问题。
最简单的方法可能是使用主机网络模式。



```
docker run --rm --name jaeger-qw --network=host \
    -e SPAN_STORAGE_TYPE=grpc-plugin \
    -e GRPC_STORAGE_SERVER=127.0.0.1:7281 \
    -p 16686:16686 \
    jaegertracing/jaeger-query:latest


```

### 运行一个简单的 Flask 应用


我们将启动一个 Flask 应用程序，该程序在每个 HTTP 调用 `http://localhost:5000/process-ip` 上执行三件事：


* 从 [https://httpbin.org/ip](https://github.com):[veee加速器](https://liuyunzhuge.com) 获取 IP 地址。
* 解析它，并使用随机休眠进行伪造处理。
* 以随机休眠时间显示它。


我们首先安装依赖项：



```
pip install flask
pip install opentelemetry-distro
pip install opentelemetry-exporter-otlp

```

`opentelemetry-distro` 包会安装 API、SDK 以及您将使用的 `opentelemetry-bootstrap` 和 `opentelemetry-instrument` 工具。


以下是我们的应用代码：



```
import random
import time
import requests

from flask import Flask

app = Flask(__name__)

@app.route("/process-ip")
def process_ip():
    body = fetch()
    ip = parse(body)
    display(ip)
    return ip

def fetch():
    resp = requests.get('https://httpbin.org/ip')
    body = resp.json()
    return body

def parse(body):
    # Sleep for a random amount of time to make the span more visible.
    secs = random.randint(1, 100) / 1000
    time.sleep(secs)

    return body["origin"]

def display(ip):
    # Sleep for a random amount of time to make the span more visible.
    secs = random.randint(1, 100) / 1000
    time.sleep(secs)

    message = f"Your IP address is `{ip}`."
    print(message)

if __name__ == "__main__":
    app.run(port=5000)

```

### 自动 Instrumentation


OpenTelemetry 提供了一个名为 `opentelemetry-bootstrap` 的工具，它可以自动为您仪器化 Python 应用程序。



```
opentelemetry-bootstrap -a install

```

现在一切就绪，我们可以运行应用了：



```
# We don't need metrics.
OTEL_METRICS_EXPORTER=none \
OTEL_TRACES_EXPORTER=console \
OTEL_SERVICE_NAME=my_app \
python my_app.py

```

通过访问 [http://localhost:5000/process\-ip](https://github.com)，您应该能在控制台看到相应的跟踪记录。


这已经很好了，但如果我们可以记录每个步骤所花费的时间、获取 HTTP 请求的状态码以及响应的内容类型，那就更好了。让我们通过手动仪器化我们的应用来实现这一点！


### 手动 Instrumentation



```
import random
import time
import requests

from flask import Flask

from opentelemetry import trace

# Creates a tracer from the global tracer provider
tracer = trace.get_tracer(__name__)

app = Flask(__name__)

@app.route("/process-ip")
@tracer.start_as_current_span("process_ip")
def process_ip():
    body = fetch()
    ip = parse(body)
    display(ip)
    return ip

@tracer.start_as_current_span("fetch")
def fetch():
    resp = requests.get('https://httpbin.org/ip')
    body = resp.json()

    headers = resp.headers
    current_span = trace.get_current_span()
    current_span.set_attribute("status_code", resp.status_code)
    current_span.set_attribute("content_type", headers["Content-Type"])
    current_span.set_attribute("content_length", headers["Content-Length"])

    return body

@tracer.start_as_current_span("parse")
def parse(body):
    # Sleep for a random amount of time to make the span more visible.
    secs = random.randint(1, 100) / 1000
    time.sleep(secs)

    return body["origin"]

@tracer.start_as_current_span("display")
def display(ip):
    # Sleep for a random amount of time to make the span more visible.
    secs = random.randint(1, 100) / 1000
    time.sleep(secs)

    message = f"Your IP address is `{ip}`."
    print(message)

    current_span = trace.get_current_span()
    current_span.add_event(message)

if __name__ == "__main__":
    app.run(port=5000)


```

我们现在可以启动新的仪器化应用：



```
OTEL_METRICS_EXPORTER=none \
OTEL_TRACES_EXPORTER=console \
OTEL_SERVICE_NAME=my_app \
opentelemetry-instrument python my_instrumented_app.py

```

如果您再次访问 [http://localhost:5000/process\-ip](https://github.com)，您应该能看到带有名称 `fetch`、`parse` 和 `display` 的新跨度以及相应的自定义属性！


### 将跟踪数据发送到 Quickwit


要将跟踪数据发送到 Quickwit，我们需要使用 OTLP 导出器。这非常简单：



```
OTEL_METRICS_EXPORTER=none \ # We don't need metrics
OTEL_SERVICE_NAME=my_app \
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:7281 \
opentelemetry-instrument python my_instrumented_app.py

```

现在，如果您访问 [http://localhost:5000/process\-ip](https://github.com)，跟踪数据将被发送到 Quickwit，只需等待大约 30 秒即可完成索引。是时候休息一下喝杯咖啡了！


30 秒过去了，让我们查询服务的跟踪数据：



```
curl -XPOST http://localhost:7280/api/v1/otel-trace-v0/search -H 'Content-Type: application/json' -d '{
    "query": "resource_attributes.service.name:my_app"
}'

```

然后打开 Jaeger UI [localhost:16686](https://github.com) 并进行操作，现在您有了一个由 Quickwit 存储后端支持的 Jaeger UI！


[![image](https://img2024.cnblogs.com/blog/436453/202408/436453-20240830134448765-1167371517.webp)](https://github.com)


[![image](https://img2024.cnblogs.com/blog/436453/202408/436453-20240830134455271-472782412.webp)](https://github.com)


### 将跟踪数据发送到您的 OpenTelemetry 收集器


按照 [OpenTelemetry 收集器教程](https://github.com) 中的说明启动一个收集器，并执行以下命令：


* [https://quickwit.io/docs/distributed\-tracing/send\-traces/using\-otel\-collector](https://github.com)



```
OTEL_METRICS_EXPORTER=none \ # We don't need metrics
OTEL_SERVICE_NAME=my_app \
opentelemetry-instrument python instrumented_app.py

```

跟踪数据将被发送到您的收集器，然后再发送到 Quickwit。


### 总结


在本教程中，我们学习了如何使用 OpenTelemetry 为 Python 应用程序进行仪器化，并将跟踪数据发送到 Quickwit。同时，我们也了解了如何使用 Jaeger UI 分析这些跟踪数据。


所有的代码片段都可在我们的 [教程仓库](https://github.com) 中找到。


* [https://github.com/quickwit\-oss/tutorials](https://github.com)


请告诉我们您对本教程的看法，如有任何疑问，欢迎通过 [Discord](https://github.com) 或 [Twitter](https://github.com) 与我们联系。


* [https://discord.gg/7eNYX4d](https://github.com)
* [https://twitter.com/quickwit\_inc](https://github.com)


## OTEL service


Quickwit 本地支持 [OpenTelemetry 协议 (OTLP)](https://github.com)，并提供了一个 gRPC 端点来接收来自 OpenTelemetry collector 或直接从应用程序通过 exporter 发送的跨度数据。此端点默认是启用的。


* [https://opentelemetry.io/docs/reference/specification/protocol/otlp/](https://github.com)


当启用时，Quickwit 将启动 gRPC 服务，准备接收来自 OpenTelemetry collector 的跨度数据。这些跨度数据默认会被索引到 `otel-trace-v0_7` 索引中，并且如果该索引不存在，它将自动创建。索引文档映射在下一个[section](https://github.com)中描述。


* [https://quickwit.io/docs/distributed\-tracing/otel\-service\#trace\-and\-span\-data\-model](https://github.com)


如果由于任何原因，您想要禁用这个端点，您可以：


* 在启动 Quickwit 时将环境变量 `QW_ENABLE_OTLP_ENDPOINT` 设置为 `false`。
* 或者[配置节点配置](https://github.com)，将索引器设置 `enable_otlp_endpoint` 设置为 `false`。
	+ [https://quickwit.io/docs/main\-branch/configuration/node\-config](https://github.com)



```
# ... Indexer configuration ...
indexer:
    enable_otlp_endpoint: false

```

### 在您选择的索引中发送跨度


您可以通过将 gRPC 请求的头部 `qw-otel-traces-index` 设置为目标索引 ID 来在您选择的索引中发送跨度。


### 跟踪和跨度数据模型


一个跟踪是一组跨度，表示一个单独的请求。一个跨度表示跟踪内的单个操作。OpenTelemetry 收集器发送跨度，Quickwit 默认将它们索引到 `otel-trace-v0_7` 索引中，该索引将 OpenTelemetry 的跨度模型映射到 Quickwit 中的索引文档。


跨度模型源自 [OpenTelemetry 规范](https://github.com)。


* [https://opentelemetry.io/docs/reference/specification/trace/api/](https://github.com)


下面是 `otel-trace-v0_7` 索引的文档映射：



```

version: 0.7

index_id: otel-trace-v0_7

doc_mapping:
  mode: strict
  field_mappings:
    - name: trace_id
      type: bytes
      input_format: hex
      output_format: hex
      fast: true
    - name: trace_state
      type: text
      indexed: false
    - name: service_name
      type: text
      tokenizer: raw
      fast: true
    - name: resource_attributes
      type: json
      tokenizer: raw
    - name: resource_dropped_attributes_count
      type: u64
      indexed: false
    - name: scope_name
      type: text
      indexed: false
    - name: scope_version
      type: text
      indexed: false
    - name: scope_attributes
      type: json
      indexed: false
    - name: scope_dropped_attributes_count
      type: u64
      indexed: false
    - name: span_id
      type: bytes
      input_format: hex
      output_format: hex
    - name: span_kind
      type: u64
    - name: span_name
      type: text
      tokenizer: raw
      fast: true
    - name: span_fingerprint
      type: text
      tokenizer: raw
    - name: span_start_timestamp_nanos
      type: datetime
      input_formats: [unix_timestamp]
      output_format: unix_timestamp_nanos
      indexed: false
      fast: true
      fast_precision: milliseconds
    - name: span_end_timestamp_nanos
      type: datetime
      input_formats: [unix_timestamp]
      output_format: unix_timestamp_nanos
      indexed: false
      fast: false
    - name: span_duration_millis
      type: u64
      indexed: false
      fast: true
    - name: span_attributes
      type: json
      tokenizer: raw
      fast: true
    - name: span_dropped_attributes_count
      type: u64
      indexed: false
    - name: span_dropped_events_count
      type: u64
      indexed: false
    - name: span_dropped_links_count
      type: u64
      indexed: false
    - name: span_status
      type: json
      indexed: true
    - name: parent_span_id
      type: bytes
      input_format: hex
      output_format: hex
      indexed: false
    - name: events
      type: array
      tokenizer: raw
      fast: true
    - name: event_names
      type: array
      tokenizer: default
      record: position
      stored: false
    - name: links
      type: array
      tokenizer: raw

  timestamp_field: span_start_timestamp_nanos

indexing_settings:
  commit_timeout_secs: 10

search_settings:
  default_search_fields: []

```

## 更多


1\. [Binance 如何使用 Quickwit 构建 100PB 日志服务(Quickwit 博客)](https://github.com)


