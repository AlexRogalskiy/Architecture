# Metrics: collection and analysis

Our tool of choice for collecting and disseminating metrics is Prometheus.

You can view the metrics generated via Prometheus by navigating (with an authenticated browser) to the `/metrics` endpoint on your favourite Octopus instance.

## Getting started

All you need to do is:

1. create a class implementing `Prometheus.DotNetRuntime.Metrics.IMetricProducer`; and
1. leave it lying around somewhere sensible.

It will be automatically registered as an Autofac `.SingleInstance()` and hooked up to the collector as appropriate.

**Please do not customise the registration of metric producers.** The intention is to make them nigh on impossible
to register incorrectly (or to forget to register), which means having a convention-based, scanning registration.

## Under the covers

As far as Octopus Server is concerned, the `/metrics` endpoint is the end of its responsibilities regarding metrics. Octopus neither knows nor cares where those metrics go. It does not push them anywhere itself, nor is there any intention for it to do so.

In Octopus Cloud, we currently run a Prometheus collector which hits the `/metrics` endpoint and forwards the metrics to Sumo Logic, but it's just as simple to hook it up to Grafana, InfluxDB or any other visualisation tool as long as there exists a Prometheus collector for it.

### When are metrics collected?

That's entirely up to the scrapers, or any other consumers of the `/metrics` endpoint.

* You should assume that metrics *may be collected very frequently*.
* You should assume that metrics *may also never be collected at all*.

### When are metrics calculated?

The `UpdateMetrics` method on your implementation of `IMetricProducer` will be called potentially every time a request is made to `/metrics`, so this should be cheap.

If a metric is not cheap to calculate, please *carefully* consider whether caching it is sensible.

### Can metric producers take dependencies?

Your `IMetricProducer` implementation may, of course, take constructor dependencies as long as those dependencies are also registered as `.SingleInstance()`.

It would generally be a good idea to avoid doing too much heavy lifting to calculate the metrics. As a general rule, **the more expensive a metric is to calculate, the more important it needs to be in order to pay for itself**.
