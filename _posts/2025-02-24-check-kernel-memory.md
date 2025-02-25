---
layout: post
title: "slabtop, to check kernel memory usage"
tags: [kubernetes]
comments: false
---

Today, I learnt about `slabtop`[^1], a command line utility to check the memory used by kernel  
(or as its man page says, it displays the kernel slab cache information in real time[^2]).

![image](https://github.com/user-attachments/assets/623dcdd3-a8e5-4d59-a802-08daed8ba619)


---

(Logging for the future me)

_**So, what led to finding this?**_ 🙂

[Jason Braganza](https://janusworx.com) taught me this, while we were preparing for our upcoming conference talk (on Kubernetes Metrics)!

Precisely, following is the metrics (exposed by `kubelet` component within a kubernetes cluster) that led to the discussion.

```
# HELP container_memory_kernel_usage Size of kernel memory allocated in bytes.
# TYPE container_memory_kernel_usage gauge
container_memory_kernel_usage{container="",id="/",image="",name="",namespace="",pod=""} 0 1732452865827
```

And how `kubelet` gets this "kernel memory allocation" information and feeds to the metrics?

So, the answer is – 

`kubelet` server package imports ["github.com/google/cadvisor/metrics"](https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L39C2-L39C38)[^3] module.

This `cadvisor/metrics` go module provides `NewPrometheusCollector()` function (which kubelet uses here[^4]).

The `NewPrometheusCollector()` function among many others, take `includedMetrics` as a parameter.

  ```
  r.RawMustRegister(metrics.NewPrometheusCollector(prometheusHostAdapter{s.host}, containerPrometheusLabelsFunc(s.host), includedMetrics, clock.RealClock{}, cadvisorOpts))
  ```

And if this `includedMetrics` contains `cadvisormetrics.MemoryUsageMetrics` (which it does, check here[^6]), 

```
	includedMetrics := cadvisormetrics.MetricSet{
		...
		cadvisormetrics.MemoryUsageMetrics:  struct{}{},
		...
	}
```

then `NewPrometheusCollector()` function exposes `container_memory_kernel_usage`[^5]  metrics


```
func NewPrometheusCollector(i infoProvider, f ContainerLabelsFunc, includedMetrics container.MetricSet, now clock.Clock, opts v2.RequestOptions) *PrometheusCollector {
  ...
  ...
	if includedMetrics.Has(container.MemoryUsageMetrics) {
		c.containerMetrics = append(c.containerMetrics, []containerMetric{
    ...
    ...
    {
				name:      "container_memory_kernel_usage",
				help:      "Size of kernel memory allocated in bytes.",
				valueType: prometheus.GaugeValue,
				getValues: func(s *info.ContainerStats) metricValues {
					return metricValues{{value: float64(s.Memory.KernelUsage), timestamp: s.Timestamp}}
				},
			},
      ...
      ...
```

And as we see above in the definition of `container_memory_kernel_usage` metrics, the `valueType` is `prometheus.Gaugevalue` (so its a gauge type metrics), and the value is `value: float64(s.Memory.KernelUsage)` where `KernelUsage` is defined here[^7] and interpreted here[^8].

I think I can go further down the rabbit hole (because I'm still not convinced of the lowest most bits, but that's all for now).

---

[^1]: which in turn gets the information from `/proc/slabinfo`.
[^2]: to me it looks something like, `top` or `htop`.
[^3]: here: [https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L39C2-L39C38](https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L39C2-L39C38)
[^4]: kubelet registring the cadvisor metrics provided by cdvisor's `metrics.NewPrometheusCollector(...)` function – https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L463
[^5]: codeblock adding `container_memory_kernel_usage` metrics: [https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/vendor/github.com/google/cadvisor/metrics/prometheus.go#L371-L393](https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/vendor/github.com/google/cadvisor/metrics/prometheus.go#L371-L393)
[^6]: Kubelet server package creating a (cadvisor metrics based) set of includedMetrics: [https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L441-L451](https://github.com/kubernetes/kubernetes/blob/d92b99ea637ee67a5c925e5e628f5816a01162ac/pkg/kubelet/server/server.go#L441-L451)
[^7]: Cadvisor's `MemoryStats` struct providing `KernelUsage`: [https://github.com/google/cadvisor/blob/5bd422f9e1cea876ee9d550f2ed95916e1766f1a/info/v1/container.go#L430-L432](https://github.com/google/cadvisor/blob/5bd422f9e1cea876ee9d550f2ed95916e1766f1a/info/v1/container.go#L430-L432)
[^8]: as part of the Cadvisor's `setMemoryStats()` function: [https://github.com/google/cadvisor/blob/5bd422f9e1cea876ee9d550f2ed95916e1766f1a/container/libcontainer/handler.go#L799-L803](https://github.com/google/cadvisor/blob/5bd422f9e1cea876ee9d550f2ed95916e1766f1a/container/libcontainer/handler.go#L799-L803)
