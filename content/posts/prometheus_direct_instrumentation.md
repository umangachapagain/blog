---
title: "Direct instrumentation with Prometheus client_golang"
description: "Instrumenting applications using direct instrumentation"
toc: true
authors: ["umangach"]
tags: ["instrumentation","prometheus", "metrics", "golang"]
categories: []
series: ["Instrumenting Applications with Prometheus client_golang"]
date: 2021-08-14T21:47:30+05:30
lastmod: 2021-08-14T21:47:30+05:30
featuredVideo:
featuredImage:
draft: false
---
If you have an application with source code in your control, you can define metrics and add
desired instrumentation inline to the code. Let's look at the required basics and write a
direct instrumented random number generator.
<!--more-->

## Interfaces involved in creating and exposing metrics

### Metric interface

```go
type Metric interface {
    Desc() *Desc
    Write(*dto.Metric) error
}
```

Source : [github.com/prometheus/client_golang](https://github.com/prometheus/client_golang/blob/2261d5cda14eb2adc5897b56996248705f9bb840/prometheus/metric.go#L29-L58)

A metric is a single value with it's metadata (i.e. metric labels) exported to Prometheus.
Counter, Gauge, Histogram and Summary implement the Metric interface.

### Collector interface

```go
type Collector interface {
    Describe(chan<- *Desc)
    Collect(chan<- Metric)
}
```

Source: [github.com/prometheus/client_golang](https://github.com/prometheus/client_golang/blob/20eef740428e9c218cf8637d9b4f4d7e3f5d4d4f/prometheus/collector.go#L16-L63)

Collectors are used by Prometheus to collect metrics. Anything that implements Collector
interface is a collector. Some default collectors provided by Prometheus are Counter,
Gauge, Histogram and Summary. These can be used for direct instrumentation of application.

### Registerer interface

```go
type Registerer interface {
    Register(Collector) error
    MustRegister(...Collector)

    Unregister(Collector) bool
}
```

Source: [github.com/prometheus/client_golang](https://github.com/prometheus/client_golang/blob/20eef740428e9c218cf8637d9b4f4d7e3f5d4d4f/prometheus/registry.go#L92-L135)

Each collector needs to register itself to a registry. You can use the default registry provided
by Prometheus or create a custom registry. These registries implement the Registerer interface.
Registries can unregister collectors if required.

### Gatherer interface

```go
type Gatherer interface {
    Gather() ([]*dto.MetricFamily, error)
}
```

Source: [github.com/prometheus/client_golang](https://github.com/prometheus/client_golang/blob/20eef740428e9c218cf8637d9b4f4d7e3f5d4d4f/prometheus/registry.go#L137-L162)

Metrics from all registered collectors are collected by a gatherer.
These gatherers implement the Gatherer interface.

## How to create metrics using these interfaces?

We start with a sample random number generator application that prints the generated number to standard output.
Then, we instrument it to provide randomly generated numbers as metrics instead of printing to standard output.

```go=
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    go func() {
        for {
            rnd := rand.Float64()
            fmt.Println(rnd)
        }
    }()
}
```

1. **Choose a metric type for the metric to be exported**

    Choosing a metric type depends on the value being exported. If value only goes up we use Counter type.<br>
    If it can go up or down arbitrarily, we use Gauge type.

    **For example**, a randomly generated number can go up or down arbitrarily. So, we choose Gauge type.

2. **Create a gauge metric using the constructor for Gauge type**

    `prometheus.NewGauge()` constructs a metric of Gauge type based on the provided `GaugeOpts`.
    GaugeOpts are used to construct metric name and metadata.

    **For example**, `randomNumber` is a metric of Gauge type. Metric name is `random_number_generated`
    and it has a constant label `app: random_number_generator`.

    ```go
    var (
        randomNumber = prometheus.NewGauge(prometheus.GaugeOpts{
            Namespace: "random",
            Subsystem: "number",
            Name:      "generated",
            Help:      "A randomly generated number",
            ConstLabels: prometheus.Labels{
                "app": "random_number_generator",
            },
        })
    )
    ```

3. **Register the collector to collect metrics**

    Since Counter, Gauge, Histogram and Summary metric types implement the Collector interface, we can register
    it for collection. We use the default prometheus registry for that.

    **For example**, `randomNumber` collector is registered to default Prometheus registry using `prometheus.MustRegister()`.
    We do this inside `init()` as we want these to be registered as soon as application starts and all variables are defined.

    ```go
    func init() {
        prometheus.MustRegister(randomNumber)
    }
    ```

4. **Set metric value**

    Each metric type provides it's own set of methods to set metric value. Gauge type provides a `Set()` method among others to
    set any arbitrary metric value.

    **For example**, we set our metric value to the randomly generated number `rnd`.

    ```go
    rnd := rand.Float64()
    randomNumber.Set(rnd)
    ```

This is what our instrumented application looks like at this point.

```go=
package main

import (
    "math/rand"

    "github.com/prometheus/client_golang/prometheus"
)

var (
    randomNumber = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "random",
        Subsystem: "number",
        Name:      "generated",
        Help:      "A randomly generated number",
        ConstLabels: prometheus.Labels{
            "app": "random_number_generator",
        },
    })
)

func init() {
    prometheus.MustRegister(randomNumber)
}

func main() {
    go func() {
        for {
            rnd := rand.Float64()
            randomNumber.Set(rnd)
        }
    }()
}
```

## How to expose metrics?

Prometheus is pull based. It means that creating metrics is not enough. We need to a way to pull (i.e. scrape) them.
We do so by exposing an http endpoint which returns the metrics in Prometheus exposition format.

1. **Create a http handler for Gatherer**

    Since Gatherer interface that calls the `Collect()` method of all registered collectors, we need an http handler
    to for it. We use [promhttp](https://github.com/prometheus/client_golang/blob/master/prometheus/promhttp/http.go)
    to create a http handler.

    ```go
    promhttp.HandlerFor(
        prometheus.DefaultGatherer,
        promhttp.HandlerOpts{},
    )
    ```

2. **Register handler to /metrics**

    Register the http handler to `/metrics`. This is the standard path to expose metrics.

    ```go
    http.Handle("/metrics", promhttp.HandlerFor(
        prometheus.DefaultGatherer,
        promhttp.HandlerOpts{},
    ))
    ```

3. **Create a listener**

    Create a listener to receive http requests. Here, we listen at port 8080 in localhost.

    ```go
    http.ListenAndServe(":8080", nil)
    ```

Finally, this is what our instrumented application looks like.

```go=
package main

import (
    "math/rand"
    "net/http"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    randomNumber = prometheus.NewGauge(prometheus.GaugeOpts{
        Namespace: "random",
        Subsystem: "number",
        Name:      "generated",
        Help:      "A randomly generated number",
        ConstLabels: prometheus.Labels{
            "app": "random_number_generator",
        },
    })
)

func init() {
    prometheus.MustRegister(randomNumber)
}

func main() {
    go func() {
        for {
            rnd := rand.Float64()
            randomNumber.Set(rnd)
        }
    }()

    http.Handle("/metrics", promhttp.HandlerFor(
        prometheus.DefaultGatherer,
        promhttp.HandlerOpts{},
    ))

    http.ListenAndServe(":8080", nil)

}
```

## Try it out

Run the application in one terminal and curl the http endpoint in another terminal to get the metrics output.

```bash
# Terminal 1
go run main.go
```

```bash
# Terminal 2
for i in {1..50};do curl -s localhost:8080/metrics|grep random_number_generated; sleep 3; echo "";done
```

Output:

```bash
# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.3726773052915178

# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.7400739387479879

# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.18367311516535434

# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.20768171508491354
```

## Sample Code

* [github.com/umangachapagain/instrumentation-example](https://github.com/umangachapagain/instrumentation-examples/blob/fb8fffa8a884e6dc43f3337468200484908b4e5a/direct-instrumentation/main.go)

## References

* [prometheus/client_golang](https://github.com/prometheus/client_golang/tree/20eef740428e9c218cf8637d9b4f4d7e3f5d4d4f)
* [Prometheus.io](https://prometheus.io/)
* [Prometheus Up and Running](https://www.oreilly.com/library/view/prometheus-up/9781492034131/)
