# [Opencensus](opencensus.io) Trace & Stats



## Trace

TBD

## Stats

### 1. Trillian servers

Each of the Trillian servers currently statically binds to `prometheus.MetricFactory` (e.g. [Log Server](https://github.com/DazWilkin/trillian/blob/72282e09c253cab36ead0c54c3835fcb0393927f/server/trillian_log_server/main.go#L88)).

To use OpenCensus, with each server, change this line (then rebuild) to:
```golang
mf := opencensus.MetricFactory{
    //Prefix: "YourDesiredPrefix",
}
```
The OpenCensus Exporter must be initialized (to connect it to an OpenCensus Agent) before use. We'll use the opportunity of initialization to parameterize the agent's endpoint:

```Golang
ocatEndpoint    = flag.String("ocat_endpoint", "localhost:55678", "Endpoint for OpenCensus Agent receiver (host:port)")
```
Then use the following initialization:
```Golang
glog.Infof("Initializing OpenCensus Agent exporter [%s]", *ocatEndpoint)
flush, err := opencensus.Initialize(*ocatEndpoint)
if err != nil {
	glog.Exitf("Failed to initialize OpenCensus Agent: %v", err)
}
defer flush()
```

The package (`"github.com/google/trillian/monitoring/opencensus"`) is already imported and so no import changes are needed.

## 2. OpenCensus Agent

The OpenCensus code exports to an OpenCensus Agent running in a separate process. By default, it expects this Agent to be listening on `:55678` but this is configurable when initializing the exporter (see above).

The following YAML (conventionally: `ocagent.yaml`) configures the Agent to:
* receive OpenCensus metric data on port `55678`
* export Stackdriver data to `{PROJECT}` (this value needs to be replaced before use) using the key referenced by `GOOGLE_APPLICATION_CREDENTIALS`
* export Prometheus data on a metrics endpoint on port `9090`.
* export Datadog data to the Datadog Agent on port `8125`. [Currently Datadog does not support metrics!]
* export zPages data on port `9999` with endpoints `/debug/rpcs` and `/debug/tracez`

```yaml
receivers:
  opencensus:
    address: ":55678"
exporters:
  stackdriver:
    project: "${PROJECT}"
    enable_metrics: true
    enable_trace: false
  prometheus:
    address: ":9090"
  # Uncertain whether Datadog is implemented correctly
  # It may only be trace-enabled presently
  datadog:
    namespace: "freddie"
    # The address:port of the Datadog Agent (:8125 is the default)
    # metrics_addr: ":8125"
    enable_metrics: true
    enable_trace: false
zpages:
  port: 9999
```
The simplest way to run the Agent is using the Docker image:
```bash
docker run \
--interactive \
--tty \
--volume=$PWD/configs:/configs \
--volume=$PWD/secrets:/secrets \
--publish=9090:9090 \
--publish=55678:55678 \
--publish=55679:55679 \
--env=GOOGLE_APPLICATION_CREDENTIALS=/secrets/key.json \
omnition/opencensus-agent:0.1.5 \
  --config=/configs/ocagent.yaml
```

**NB** The above command assumes the existence of `${PWD}/configs/ocagent.yaml` (see example above) and `${PWD}/secrets/key.json` (see 'Stackdriver' below)

## 3. Monitoring
### 3.1. zPages

* http://localhost:9999/debug/rpcz
* http://localhost:9999/debug/tracez

### 3.2. Stackdriver

```bash
PROJECT=[[YOUR-PROJECT]]

# OpenCensus metrics are represented by Stackdriver custom metrics
# Stackdriver requires billing be enabled for custom metrics
BILLING=[[YOUR-BILLING]]

gcloud projects create ${PROJECT}
gcloud beta billing projects link ${PROJECT} --billing-account=${BILLING}


# Create a service account with minimum permissions for creating|writing metrics
ACCOUNT=opencensus
FILE="${PWD}/${ACCOUNT}.key.json"

gcloud iam service-accounts create ${ACCOUNT} \
--display-name=${ACCOUNT} \
--project=${PROJECT}

gcloud iam service-accounts keys create ${FILE} \
--iam-account=${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com \
--project=${PROJECT}

# To permit the service account to write to Stackdriver Monitoring:

gcloud projects add-iam-policy-binding ${PROJECT} \
--member=serviceAccount:${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com \
--role=roles/monitoring.metricWriter

# To permit the service account to write to Stackdriver Trace:

gcloud projects add-iam-policy-binding ${PROJECT} \
--member=serviceAccount:${ACCOUNT}@${PROJECT}.iam.gserviceaccount.com \
--role=roles/cloudtrace.agent

# Stackdriver must be provisioned through the UI
google-chrome https://console.cloud.google.com/monitoring?project=${PROJECT}
```

Once Stackdriver is provisioned, you should be able to return to it:
```bash
google-chrome https://app.google.stackdriver.com/?project=${PROJECT}
```

#### 3.2.1 Console
Observe the metrics using Stackdriver Console, e.g. Metrics Explorer

**NB** 

If a Prefix is defined, the metric types will be `custom.googleapis.com/opencensus/{{MetricFactory.Prefix}}{{separator}}{{name}}`

So, for a `Prefix: "Freddie"`, the Trillian metric name `mysql_queue_leaf_latency_entry` will be `custom.googleapis.com/opencensus/Freddie_mysql_queue_leaf_leatency_entry`

Stackdriver will display this name as `OpenCensus/Freddie_mysql_queue_leaf_latency_entry`

If no Preix is define, metrics will be of the form `custom.googleapis.com/opencensus/{{name}}`

#### 3.2.2. APIs Explorer

APIs Explorer is a powerful tool and it's useful to be able to browse Stackdriver metric types and their time-series data:

For the list of metrics:

* stackdriver/monitoring/v3/monitoring.projects.metricDescriptors.list
* name: `projects/[[YOUR-PROJECT]]`
* filter: `metric.type=starts_with("custom.googleapis.com/opencensus/{{MetricFactory.Prefix}}`
* [optional] fields: `metricDescriptors/name`

For the time-series for a specific metric, e.g. `log_rpc_requests`

* stackdriver/monitoring/v3/monitoring.projects.timeSeries.list
* name: `projects/[[YOUR-PROJECT]]`
* filter: `metric.type="custom.googleapis.com/opencensus/{{MetricFactory.Prefix}}_log_rpc_requests"`
* interval.endTime: e.g. `2019-04-04T23:59:59-08:00` (4th April 2019 23:59:59 in Pacific)
* interval.startTime: e.g. `2019-04-04T00:00:00-08:00`
* [optional] fields: `timeSeries/points/value`

### 3.3. Prometheus

Easy: http://localhost:9090/metrics

### 3.4. Datadog

Run the Datadog Agent:
```bash
DD_API_KEY=[[YOUR-DATADOG-API-KEY]]
docker run \
--volume=/var/run/docker.sock:/var/run/docker.sock:ro \
--volume=/proc/:/host/proc/:ro \
--volume=/sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
--env=DD_API_KEY=${DD_API_KEY} \
--env=DD_DOGSTATSD_NON_LOCAL_TRAFFIC=true \
--publish=8125:8125/udp \
datadog/agent:latest
```

You may use Datadog's console to observe metrics being exported.


## 4. Tests.

OpenCensus is a one-way proxy to one or more monitoring services; it is unable to query values for metrics that it has forwarded. For this reason, the tests which all depend upon `Value` fail:

```bash
go test github.com/google/trillian/monitoring/opencensus
```
Errors are to expected:
```bash

--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/counter0 (0.00s)
        metrics.go:54: Counter[[]].Value()=0; want 1
        metrics.go:58: Counter[[]].Value()=0; want 3.5
        metrics.go:69: Counter[[]].Value()=0; want 3.5
    --- FAIL: TestCounter/counter1 (0.00s)
        metrics.go:54: Counter[[val1]].Value()=0; want 1
        metrics.go:58: Counter[[val1]].Value()=0; want 3.5
        metrics.go:69: Counter[[val1]].Value()=0; want 3.5
    --- FAIL: TestCounter/counter2 (0.00s)
        metrics.go:54: Counter[[val1 val2]].Value()=0; want 1
        metrics.go:58: Counter[[val1 val2]].Value()=0; want 3.5
        metrics.go:69: Counter[[val1 val2]].Value()=0; want 3.5

--- FAIL: TestGauge (0.00s)
    --- FAIL: TestGauge/gauge0 (0.00s)
        metrics.go:106: Gauge[[]].Value()=0; want 1
        metrics.go:114: Gauge[[]].Value()=0; want 2.5
        metrics.go:118: Gauge[[]].Value()=0; want 42
        metrics.go:132: Gauge[[]].Value()=0; want 42
    --- FAIL: TestGauge/gauge1 (0.00s)
        metrics.go:106: Gauge[[val1]].Value()=0; want 1
        metrics.go:114: Gauge[[val1]].Value()=0; want 2.5
        metrics.go:118: Gauge[[val1]].Value()=0; want 42
        metrics.go:132: Gauge[[val1]].Value()=0; want 42
    --- FAIL: TestGauge/gauge2 (0.00s)
        metrics.go:106: Gauge[[val1 val2]].Value()=0; want 1
        metrics.go:114: Gauge[[val1 val2]].Value()=0; want 2.5
        metrics.go:118: Gauge[[val1 val2]].Value()=0; want 42
        metrics.go:132: Gauge[[val1 val2]].Value()=0; want 42

--- FAIL: TestHistogram (0.00s)
    --- FAIL: TestHistogram/histogram0 (0.00s)
        metrics.go:173: Histogram[[]].Info()=0,0; want 3,6
        metrics.go:187: Histogram[[]].Info()=0,0; want 3,6
    --- FAIL: TestHistogram/histogram1 (0.00s)
        metrics.go:173: Histogram[[val1]].Info()=0,0; want 3,6
        metrics.go:187: Histogram[[val1]].Info()=0,0; want 3,6
    --- FAIL: TestHistogram/histogram2 (0.00s)
        metrics.go:173: Histogram[[val1 val2]].Info()=0,0; want 3,6
        metrics.go:187: Histogram[[val1 val2]].Info()=0,0; want 3,6
FAIL
FAIL	github.com/google/trillian/monitoring/opencensus	0.077s
```