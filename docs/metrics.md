# TorchServe Metrics

## Contents of this document

* [Introduction](#introduction)
* [System metrics](#system-metrics)
* [Formatting](#formatting)
* [Metric Types](#metric-types)
* [Central metrics yaml file definition](#central-metrics-yaml-file-definition)
* [Custom Metrics API](#custom-metrics-api)
* [Log custom metrics](#log-custom-metrics)
* [Metrics YAML Parsing and Metrics API example](#Metrics-YAML-File-Parsing-and-Metrics-API-Custom-Handler-Example)

## Introduction

TorchServe collects system level metrics in regular intervals, and also provides an API to collect custom metrics.
Metrics collected by metrics are logged and can be aggregated by metric agents.
The system level metrics are collected every minute. Metrics defined by the custom service code can be collected per request or per a batch of requests.
TorchServe logs these two sets of metrics to different log files.
Metrics are collected by default at:

* System metrics - `log_directory/ts_metrics.log`
* Custom metrics - `log directory/model_metrics.log`

The location of log files and metric files can be configured in the [log4j2.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/resources/log4j2.xml) file

## System Metrics

| Metric Name | Dimension | Unit | Semantics |
|---|---|---|---|
| CPUUtilization | host | percentage | CPU utilization on host |
| DiskAvailable | host | GB | disk available on host |
| DiskUsed | host | GB | disk used on host |
| DiskUtilization | host | percentage | disk used on host |
| MemoryAvailable | host | MB | memory available on host |
| MemoryUsed | host | MB | memory used on host |
| MemoryUtilization | host | percentage | memory utilization on host |
| GPUUtilization | host,device_id | percentage | GPU utilization on host,device_id |
| GPUMemoryUtilization | host,device_id | percentage | GPU memory utilization on host,device_id |
| GPUMemoryUsed | host,device_id | MB | GPU memory used on host,device_id |
| Requests2XX | host | count | logged for every request responded in 200-300 status code range |
| Requests4XX | host |count | logged for every request responded in 400-500 status code range |
| Requests5XX | host | count | logged for every request responded with status code above 500 |

## Formatting

TorchServe emits metrics to log files by default. The metrics are formatted in a [StatsD](https://github.com/etsy/statsd) like format.

```bash
CPUUtilization.Percent:0.0|#Level:Host|#hostname:my_machine_name
MemoryUsed.Megabytes:13840.328125|#Level:Host|#hostname:my_machine_name
```

To enable metric logging in JSON format, set "patternlayout" as "JSONPatternLayout" in [log4j2.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/resources/log4j2.xml) (See sample [log4j2-json.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/test/resources/log4j2-json.xml)). For information, see [Logging in Torchserve](https://github.com/pytorch/serve/blob/master/docs/logging.md).

After you enable JSON log formatting, logs will look as follows:

```json
{
  "MetricName": "DiskAvailable",
  "Value": "108.15547180175781",
  "Unit": "Gigabytes",
  "Dimensions": [
    {
      "Name": "Level",
      "Value": "Host"
    }
  ],
  "HostName": "my_machine_name"
}
```

```json
{
  "MetricName": "DiskUsage",
  "Value": "124.13163757324219",
  "Unit": "Gigabytes",
  "Dimensions": [
    {
      "Name": "Level",
      "Value": "Host"
    }
  ],
  "HostName": "my_machine_name"
}

```

To enable metric logging in QLog format, set "patternlayout" as "QLogLayout" in [log4j2.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/resources/log4j2.xml) (See sample [log4j2-qlog.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/test/resources/log4j2-qlog.xml)). For information, see [Logging in Torchserve](https://github.com/pytorch/serve/blob/master/docs/logging.md).

After you enable QLogsetupModelDependencies formatting, logs will look as follows:

```qlog
HostName=abc.com
StartTime=1646686978
Program=MXNetModelServer
Metrics=MemoryUsed=5790.98046875 Megabytes Level|Host
EOE
HostName=147dda19895c.ant.amazon.com
StartTime=1646686978
Program=MXNetModelServer
Metrics=MemoryUtilization=46.2 Percent Level|Host
EOE

```

## Metric Types

TorchServe Metrics is introducing [Metric Types](https://github.com/pytorch/serve/blob/master/ts/metrics/metric_type_enums.py)
that are in line with the [Prometheus API](https://github.com/prometheus/client_python) metric types.

Metric types are an attribute of Metric objects.
Users will be restricted to the existing metric types when adding metrics via Metrics API.

```python
class MetricTypes(enum.Enum):
    counter = "counter"
    gauge = "gauge"
    histogram = "histogram"
```

## Central metrics YAML file definition

TorchServe defines metrics in a [metrics_default.yaml](https://github.com/pytorch/serve/blob/master/frontend/server/src/test/resources/metrics_default.yaml)
file, including both frontend metrics (i.e. `ts_metrics`) and backend metrics (i.e. `model_metrics`).
When TorchServe is started, the metrics definition is loaded in the frontend and backend cache separately.
The backend flushes the metrics cache once a load model or inference request is completed.

Dynamic updates between the frontend and backend are _not_ currently being handled.

The `metrics.yaml` is formatted with Prometheus metric type terminology:

```yaml
dimensions:  # dimensions dict
  ModelName: "ExampleModelName"
  Level: "Model"
  host: "example_host_name"

ts_metrics:  # frontend metrics
  counter:  # metric type
    - NameOfCounterMetric:  # name of metric
        unit: ms  # unit of metric
        dimensions: [ModelName, host]  # dimensions of metric (referenced from the above dimensions dict)
  gauge:
    - NameOfGaugeMetric:
        unit: ms
        dimensions: [ModelName, host]
  histogram:
    - NameOfHistogramMetric:
        unit: ms
        dimensions: [ModelName, host]

model_metrics:  # backend metrics
  counter:  # metric type
    - InferenceTimeInMS:  # name of metric
        unit: ms  # unit of metric
        dimensions: [ModelName, Level]  # dimensions of metric (referenced from the above dimensions dict)
    - NumberOfMetrics:
        unit: count
        dimensions: [model_name]
  gauge:
    - GaugeModelMetricNameExample:
        unit: ms
        dimensions: [ModelName, Level]
  histogram:
    - HistogramModelMetricNameExample:
        unit: ms
        dimensions: [ModelName, Level]
```


These are the default metrics within the yaml file, but the user can either delete them to their liking / ignore them altogether, because these metrics will not be emitted
unless they are edited.


### How it works

Whenever torchserve starts, the [model_loader](https://github.com/pytorch/serve/blob/master/ts/model_loader.py) initializes
`service.context.metrics` with the [MetricsCache](https://github.com/pytorch/serve/blob/master/ts/metrics/metric_cache_yaml.py) object.
The `model_metrics` (backend metrics) section within the specified yaml file will be parsed,
and Metric objects will be created based on the parsed section and added  that are added to the cache.

This is all done internally, so the user does not have to do anything other than specifying the desired yaml file.

*Users have the ability to parse other sections of the yaml file manually, but the primary purpose of this functionality is to
parse the backend metrics from the yaml file.

### User Manual - starting TorchServe with a yaml file specified

1. Create a `metrics.yaml` file to parse metrics from OR utilize default [metrics_default.yaml](https://github.com/pytorch/serve/blob/master/frontend/server/src/test/resources/metrics_default.yaml)


2. Set `metrics_config` argument equal to the yaml file path in the `config.properties` being used:
    ```properties
    ...
    ...
    # enable_metrics_api=false
    workflow_store=../archive/src/test/resources/workflows
    metrics_config=../../../ts/tests/unit_tests/metrics_yaml_testing/metrics.yaml
    ...
    ...
    ```

   If a `metrics_config` argument is not specified, the default yaml file will be used.


3. Run torchserve and specify the path of the `config.properties` after the `ts-config` flag:
   (example using [Huggingface_Transformers](https://github.com/pytorch/serve/tree/master/examples/Huggingface_Transformers))

   ```torchserve --start --model-store model_store --models my_tc=BERTSeqClassification.mar --ncs --ts-config ../../frontend/server/src/test/resources/config.properties```


## Custom Metrics API

TorchServe enables the custom service code to emit metrics that are then logged by the system.

The custom service code is provided with a [context](https://github.com/pytorch/serve/blob/master/ts/context.py) of the current request with a metrics object:


```python
# Access context metrics as follows
metrics = context.metrics
```

All metrics are collected within the context.


### Specifying Metric Types

When adding any metric via Metrics API, users have the ability to override the metric type by specifying the positional argument
`metric_type=MetricTypes.[counter/gauge/histogram]`.

```python
metrics.add_metric("GenericMetric", value=1, ..., metric_type=MetricTypes.gauge)

metrics.add_counter("CounterMetric", value=1, ..., metric_type=MetricTypes.histogram)
```


### Updating Metrics parsed from the yaml file
Given the Metrics API, users will also be able to update metrics that have been parsed from the [yaml](https://github.com/pytorch/serve/blob/master/frontend/server/src/test/resources/metrics_default.yaml) file
given some criteria:

(we will use the following metric as an example)
```yaml
  counter:  # metric type
    - InferenceTimeInMS:  # name of metric
        unit: ms  # unit of metric
        dimensions: [ModelName, Level]
```
1. Metric Type has to be the same
   1. The user will have to use a counter-based `add_...` method,
      or explicitly set `metric_type=MetricTypes.counter` within the `add_...` method


2. Metric Name has to be the same
   1. If the name of the metric in the YAML file you want to update is `InferenceTimeInMS`, then `add_metric(name="InferenceTimeInMS", ...)`


3. Dimensions should be the same (as well as the same order!)
   1. All dimensions have to match, and Metric objects that have been parsed from the yaml file also have dimensions that are parsed from the yaml file
      1. Users can [create their own](#create-dimension-objects) `Dimension` objects to match those in the yaml file dimensions
      2. if the Metric object has `ModelName` and `Level` dimensions only, it is optional to specify additional dimensions since these are
      considered [default dimensions](#default-dimensions), so: `add_counter('InferenceTimeInMS', value=2)` or `add_counter('InferenceTimeInMS', value=2, dimensions=["ModelName", "Level"])`
      3. if Metric object has additional dimension, just specify the additional dimensions within the API
         1. if metric in yaml had `dimensions: [foo, ModelName, Level]`, then `add_counter(..., dimensions=["foo"])`
         2. `dimensions: [foo, ModelName, Level]` != `dimensions: [ModelName, Level, foo]`


### Default dimensions

Metrics will have a couple of default dimensions if not already specified.

If the metric is a type `Gauge`, `Histogram`, `Counter`, by default it will have:
  * `ModelName,{name_of_model}`
  * `Level,Model`


### Create dimension object(s)

Dimensions for metrics can be defined as objects

```python
from ts.metrics.dimension import Dimension

# Dimensions are name value pairs
dim1 = Dimension(name, value)
dim2 = Dimension(some_name, some_value)
.
.
.
dimN= Dimension(name_n, value_n)

```

**NOTE:** Metric functions below accept a list of dimensions

### Add generic metrics

**Generic metrics are defaulted to a `counter` metric type**

One can add metrics with generic units using the following function.

Function API

```python
    def add_metric(self, metric_name: str, value: int or float, unit: str, idx=None, dimensions: list = None,
                   metric_type: MetricTypes = MetricTypes.counter) -> None:
        """
        Create a new metric and add into cache.
            Add a metric which is generic with custom metrics

        Parameters
        ----------
        metric_name: str
            Name of metric
        value: int, float
            value of metric
        unit: str
            unit of metric
        idx: int
            request_id index in batch
        dimensions: list
            list of dimensions for the metric
        metric_type: MetricTypes
            Type of metric
        """
```

```python
# Add Distance as a metric
# dimensions = [dim1, dim2, dim3, ..., dimN]
# Assuming batch size is 1 for example
metrics.add_metric('DistanceInKM', distance, 'km', dimensions=dimensions)
```

### Add time-based metrics

**Time-based metrics are defaulted to a `gauge` metric type**

Add time-based by invoking the following method:

Function API

```python
    def add_time(self, name: str, value: int or float, idx=None, unit: str = 'ms', dimensions: list = None,
                 metric_type: MetricTypes = MetricTypes.gauge):
        """
        Add a time based metric like latency, default unit is 'ms'
            Default metric type is gauge

        Parameters
        ----------
        name : str
            metric name
        value: int
            value of metric
        idx: int
            request_id index in batch
        unit: str
            unit of metric,  default here is ms, s is also accepted
        dimensions: list
            list of dimensions for the metric
        metric_type: MetricTypes
           type for defining different operations, defaulted to gauge metric type for Time metrics
        """
```

Note that the default unit in this case is 'ms'

**Supported units**: `['ms', 's']`

To add custom time-based metrics:

```python
# Add inference time
# dimensions = [dim1, dim2, dim3, ..., dimN]
# Assuming batch size  is 1 for example
metrics.add_time('InferenceTime', end_time-start_time, None, 'ms', dimensions)
```

### Add size-based metrics

**Size-based metrics are defaulted to a `gauge` metric type**

Add size-based metrics by invoking the following method:

Function API

```python
    def add_size(self, name: str, value: int or float, idx=None, unit: str = 'MB', dimensions: list = None,
                 metric_type: MetricTypes = MetricTypes.gauge):
        """
        Add a size based metric
            Default metric type is gauge

        Parameters
        ----------
        name : str
            metric name
        value: int, float
            value of metric
        idx: int
            request_id index in batch
        unit: str
            unit of metric, default here is 'MB', 'kB', 'GB' also supported
        dimensions: list
            list of dimensions for the metric
        metric_type: MetricTypes
           type for defining different operations, defaulted to gauge metric type for Size metrics
        """
```

Note that the default unit in this case is milliseconds (ms).

**Supported units**: `['MB', 'kB', 'GB']`

To add custom size based metrics

```python
# Add Image size as a metric
# dimensions = [dim1, dim2, dim3, ..., dimN]
# Assuming batch size 1
metrics.add_size('SizeOfImage', img_size, None, 'MB', dimensions)
```

### Add Percentage based metrics

**Percentage-based metrics are defaulted to a `gauge` metric type**

Percentage based metrics can be added by invoking the following method:

Function API

```python
    def add_percent(self, name: str, value: int or float, idx=None, dimensions: list = None,
                    metric_type: MetricTypes = MetricTypes.gauge):
        """
        Add a percentage based metric
            Default metric type is gauge

        Parameters
        ----------
        name : str
            metric name
        value: int, float
            value of metric
        idx: int
            request_id index in batch
        dimensions: list
            list of dimensions for the metric
        metric_type: MetricTypes
           type for defining different operations, defaulted to gauge metric type for Percent metrics
        """

```

To add custom percentage-based metrics:

```python
# Add MemoryUtilization as a metric
# dimensions = [dim1, dim2, dim3, ..., dimN]
# Assuming batch size 1
metrics.add_percent('MemoryUtilization', utilization_percent, None, dimensions)
```

### Add counter-based metrics

**Counter-based metrics are defaulted to a `counter` metric type**

Counter based metrics can be added by invoking the following method

Function API

```python
    def add_counter(self, name: str, value: int or float, idx=None, dimensions: list = None,
                    metric_type: MetricTypes = MetricTypes.counter):
        """
        Add a counter metric or increment an existing counter metric
            Default metric type is counter
        Parameters
        ----------
        name : str
            metric name
        value: int or float
            value of metric
        idx: int
            request_id index in batch
        dimensions: list
            list of dimensions for the metric
        metric_type: MetricTypes
           type for defining different operations, defaulted to counter metric type for Counter metrics
        """
```

To create, increment and decrement counter-based metrics we can use the following calls:

```python
# Add Loop Count as a metric
# dimensions = [dim1, dim2, dim3, ..., dimN]
# Assuming batch size 1

# Create a counter with name 'LoopCount' and dimensions, initial value
metrics.add_counter('LoopCount', 1, None, dimensions)

# Increment counter by 2
metrics.add_counter('LoopCount', 2 , None, dimensions)

# Decrement counter by 1
metrics.add_counter('LoopCount', -1, None, dimensions)

# Final counter value in this case is 2

```

### Getting a metric

Users can [get a metric](https://github.com/pytorch/serve/tree/master/ts/metrics/metric_cache_yaml.py#L202)
from the cache. The Metric object is returned, so the user can access the methods of the Metric:
(i.e. `Metric.update(value)`, `Metric.__str__`)

```python
    def get_metric(self, metric_type: MetricTypes, metric_name: str, dimensions: list or str) -> Metric:
        """
        Get a Metric from cache.
            Ask user for required requirements to form metric key to retrieve Metric.

        Parameters
        ----------
        metric_type: MetricTypes
            Type of metric: use MetricTypes enum to specify

        metric_name: str
            Name of metric

        dimensions: list or str
            list of dimension keys which should be strings

        """
```

i.e.
```python
# Method 1: Getting metric of MetricType counter, metric name string, and list of dimension keys derived from YAML file.
metrics.get_metric(MetricTypes.counter, "MetricName", ["ModelName", "Level"])

# Method 2: Getting metric of MetricType gauge, metric name string, and complete log string of dimensions.
metrics.get_metric(MetricTypes.gauge, "GaugeMetricName", "ModelName:foo,Level:Model")
```


## Log custom metrics

Following sample code can be used to log the custom metrics created in the model's custom handler:

_Metrics will only be emitted if they have value assigned to them.
In other words, if metrics have an initial value of 0 (which is the default value), the metric will not be emitted._

```python
# In Custom Handler
from ts.service import emit_metrics

class ExampleCustomHandler(BaseHandler, ABC):
   def initialize(self, ctx):

emit_metrics(metrics.cache)
```

This custom metrics information is logged in the `model_metrics.log` file configured through [log4j2.xml](https://github.com/pytorch/serve/blob/master/frontend/server/src/main/resources/log4j2.xml) file.

**NOTE:** After emitting the metrics, all metric values that have been updated will be reset to 0.

## Metrics YAML File Parsing and Metrics API Custom Handler Example

This example utilizes the feature of parsing metrics from a YAML file, adding and updating metrics and their values via Metrics API,
updating metrics that have been parsed from the YAML file via Metrics API, and finally emitting all metrics that have been updated.

```python
from ts.service import emit_metrics
from ts.metrics.metric_type_enum import MetricTypes


class CustomHandlerExample:
    def initialize(self, ctx):
        metrics = ctx.metrics  # initializing metrics to the context.metrics

        # Setting a sleep for examples' sake
        start_time = time.time()
        time.sleep(3)
        stop_time = time.time()

        # Adds a metric that has a metric type of gauge
        metrics.add_time(
            "HandlerTime", round((stop_time - start_time) * 1000, 2), None, "ms"
        )

        # The next two lines will accumulate - the end result is one Metric
        # name HandlerSeparateCounter with a value of 1.2
        metrics.add_counter("HandlerSeparateCounter", 2.5)
        metrics.add_counter("HandlerSeparateCounter", -1.3)

        # Adding a standard counter metric
        metrics.add_counter("HandlerCounter", 21.3)

        # Assume that a metric that has a metric type of counter
        # and is named InferenceTimeInMS in the metrics.yaml file.
        # Instead of creating a new object with the same name and same parameters,
        # this line will update the metric that already exists from the YAML file.
        metrics.add_counter("InferenceTimeInMS", 2.78)

        # Another method of updating values -
        # using the get_metric + Metric.update method
        # In this example, we are getting an already existing
        # Metric that had been parsed from the yaml file
        histogram_example_metric = metrics.get_metric(
            MetricTypes.histogram,
            "HistogramModelMetricNameExample",
            ["ModelName", "Level"],
        )
        histogram_example_metric.update(4.6)

        # Same idea as the 'metrics.add_counter('InferenceTimeInMS', 2.78)' line,
        # except this time with gauge metric type object
        metrics.add_size("GaugeModelMetricNameExample", 42.5)

        # Emitting the metrics that have been updated to the frontend and
        # then resetting the Metrics' values afterwards
        emit_metrics(metrics.cache)
```
