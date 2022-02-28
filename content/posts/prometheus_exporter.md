---
title: "Metrics Exporter with Prometheus client_golang"
description: "Exporting Prometheus metrics from an application"
toc: true
authors: ["umangach"]
tags: ["instrumentation","prometheus", "metrics", "golang"]
categories: []
series: ["Instrumenting Applications with Prometheus client_golang"]
date: 2022-02-28T10:34:16+05:30
lastmod: 2022-02-28T10:34:16+05:30
featuredVideo:
featuredImage:
draft: false
---
In cases where [instrumenting an application with Prometheus metrics directly](/posts/prometheus_direct_instrumentation) is not feasible or when other metric formats needs to be converted to Prometheus exposition format, exporter can be used.

Example: [node_exporter](https://github.com/prometheus/node_exporter/tree/e3a18fdd37447992c213f4) which has multiple custom collectors for hardware and OS metrics exposed by *NIX kernels.
<!--more-->

## Custom Collectors
Custom collectors are custom implementation of Collector interface unlike Counter, Gauge etc.
They are ideal for converting existing data formats to Prometheus expostion format during collection.

Example: [Node Collector](https://github.com/prometheus/node_exporter/blob/e3a18fdd37447992c213f42d61fdaed997fe9351/collector/collector.go#L78-L82) and [CPU Collector](https://github.com/prometheus/node_exporter/blob/e3a18fdd37447992c213f42d61fdaed997fe9351/collector/cpu_linux.go#L33-L48) in [node_exporter](https://github.com/prometheus/node_exporter/tree/e3a18fdd37447992c213f4).

### Custom collectors in Prometheus client_golang package
The Prometheus client_golang package provides some commonly used custom collectors such as [Go collector](https://github.com/prometheus/client_golang/blob/1f81b3e9130fe21c55b7db82ecb2d61477358d61/prometheus/go_collector.go#L212-L218) and [Process collector](https://github.com/prometheus/client_golang/blob/1f81b3e9130fe21c55b7db82ecb2d61477358d61/prometheus/process_collector.go#L25-L34).

## How to write an exporter for an application?
Let us assume that the `math/rand` golang library is a random number generator application that does not expose any Prometheus metrics.

We wish to expose randomly generated numbers as metrics but we can not change the source code. So, directly instrumenting it is not possible.

Instead, we will use custom collectors to convert it's output to Prometheus exposition format. We will use the Prometheus client_golang package for this.

Follow the steps given below.

1. Define a custom type which will implement the prometheus.Collector interface.
```go=
type randomNumber struct {
	randomNumber *prometheus.Desc
}
```
`randomNumber` is a custom collector. It converts random number to Prometheus metrics.

2. Create a descriptor used by Prometheus to store metric metadata.
```go=
var (
	randomNumberDesc = prometheus.NewDesc(
		prometheus.BuildFQName("random", "number", "generated"),
		"A randomly generated number",
		nil,
		prometheus.Labels{
			"app": "random_number_generator",
		},
	)
)
```
`prometheus.NewDesc()` takes metrics name, help text, variable and constant labels as arguments and creates a `prometheus.Desc{}`. This descriptor is used by prometheus to store immutable metrics metadata. The number of labels used defines the number of timeseries to be collected (more on this later). 

3. Implement prometheus.Collector interface
```go=
func (r randomNumber) Describe(ch chan<- *prometheus.Desc) {
	ch <- r.randomNumber
}

func (r randomNumber) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(randomNumberDesc, prometheus.GaugeValue, rand.Float64())
}
```
`Describe()` sends all descriptors of metrics collected by `randomNumber` collector to the channel.

`Collect()` is called by Prometheus registry when collecting metrics. Metrics is sent via the provided channel.

Both these methods are required to satisfy the `prometheus.Collector` interface.

4. Register custom collector to Prometheus registry.
```go=
func main() {
	collector := randomNumber{
		randomNumber: randomNumberDesc,
	}
	prometheus.MustRegister(collector)
}
```
`collector` is an instance of `randomNumber` collector registered to the default Prometheus registry.

5. Expose metrics via HTTP
```go=
var (
	addr = flag.String("listen-address", ":8080", "The address to listen on for HTTP requests.")
)

func main() {
	flag.Parse()

	http.Handle("/metrics", promhttp.HandlerFor(
		prometheus.DefaultGatherer,
		promhttp.HandlerOpts{},
	))
	log.Printf("Running server at %s\n", *addr)
	log.Fatal(http.ListenAndServe(*addr, nil))
}
```

This is what complete code looks like,
```go=
package main

import (
	"flag"
	"log"
	"math/rand"
	"net/http"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

type randomNumber struct {
	randomNumber *prometheus.Desc
}

var (
	randomNumberDesc = prometheus.NewDesc(
		prometheus.BuildFQName("random", "number", "generated"),
		"A randomly generated number",
		nil,
		prometheus.Labels{
			"app": "random_number_generator",
		},
	)
    addr = flag.String("listen-address", ":8080", "The address to listen on for HTTP requests.")
)

func (r randomNumber) Describe(ch chan<- *prometheus.Desc) {
	ch <- r.randomNumber
}

func (r randomNumber) Collect(ch chan<- prometheus.Metric) {
	ch <- prometheus.MustNewConstMetric(randomNumberDesc, prometheus.GaugeValue, rand.Float64())
}

func main() {
	flag.Parse()

	collector := randomNumber{
		randomNumber: randomNumberDesc,
	}
	prometheus.MustRegister(collector)

	http.Handle("/metrics", promhttp.HandlerFor(
		prometheus.DefaultGatherer,
		promhttp.HandlerOpts{},
	))
	log.Printf("Running server at %s\n", *addr)
	log.Fatal(http.ListenAndServe(*addr, nil))
}
```

## Try it out
Run the exporter in one terminal and curl the http endpoint in another terminal to get the metrics output.

```shell=
# Terminal 1
go run main.go

2022/02/25 14:28:03 Running server at :8080
```

```shell=
# Terminal 2
for i in {1..10};do curl -s localhost:8080/metrics|grep random_number_generated; sleep 2; echo "";done

# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.6046602879796196
# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.9405090880450124
# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.6645600532184904
# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.4377141871869802
# HELP random_number_generated A randomly generated number
# TYPE random_number_generated gauge
random_number_generated{app="random_number_generator"} 0.4246374970712657
```

## Recommendations for running exporter
+ Deploy exporters along side the application to keep the architecture similar to direct instrumentation.
+ Exporters should not perform scrapes on it's own. Instead, metrics should be pulled when Prometheus scrapes targets based on it's configuration.
+ Failed scrape from collector should return 5xx errors.

## Source Code
+ [github.com/umangachapagain/instrumentation-examples](https://github.com/umangachapagain/instrumentation-examples/tree/main/exporter)

## References
+ [Prometheus.io](https://prometheus.io/)
+ [Prometheus Up and Running](https://www.oreilly.com/library/view/prometheus-up/9781492034131/)
+ [prometheus/client_golang](https://github.com/prometheus/client_golang/)
+ [node_exporter](https://github.com/prometheus/node_exporter/)