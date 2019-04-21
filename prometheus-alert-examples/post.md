[Prometheus](https://prometheus.io/) is a fantastic, open-source tool for monitoring and alerting. Besides collecting metrics from the whole system (e.g. a kubernetes cluster, or just a single instance), it's also possible to trigger alerts using the [alertmanager](https://github.com/prometheus/alertmanager).

When setting alerts up, however, I had a hard time finding concrete examples of alerts for basic things like `high cpu usage`, or for kubernetes-specific things like a pod having been restarted, so here are some  of the alerts we came up with and which, albeit with different values and sometimes in different ways, could be used in a production setting.

Some of the alerts depend on the [node-exporter](https://github.com/prometheus/node_exporter) running on the target instance(s), while others use [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) and kubernetes. There are also alerts using custom metrics for http response status and http response times.

At the end of the day, the general concept of the alerts should work regardless of the exact data source.

Let's look at some examples for alerts, which might be useful in a distributed system (especially using kubernetes).

## Infrastructure Alerts

To start off, we'll look at some basic infrastructure alerts, regarding CPU/memory/disk. We'll go over the alerts one by one, with a short explanation for each one.

To signal, that a target (e.g. instance) is down, we simply check the `up` metric:

```yml
expr: up == 0
```

To signal, that a disk will fill up soon, based on a trend of the last x hours, we use the `predict_linear` function. This alert triggers, if based on data during the last 24 hours, the available disk space will go below zero in the next 7 days:

```yml
expr: predict_linear(node_filesystem_avail[24h], 7*24*3600) <= 0
```

To signal, that memory will be full in a certain timeframe based on a trend of the last x hours, we also use `predict_linear`. This alert will trigger, if the memory increase based on the last two hours, will result in the memory running out within the next hour:

```yml
expr: predict_linear(node_memory_MemAvailable[2h], 1*3600) <= 0
```

To signal high memory pressure, we first calculate the percentage of available memory, and if that's as low as 5% and the rate of page faults during the last minute was high (>100), we trigger an alert:

```yml
expr: (node_memory_MemAvailable /  node_memory_MemTotal * 100) < 5 and rate(node_vmstat_pgmajfault[1m]) > 100
```

To signal high CPU usage, we simply divide the load average of the last 5 minutes by the amount of cpus on that instance, and if it's above 95% for some time, we alert:

```yml
expr: node_load5/count(node_cpu{mode="idle"}) without (cpu,mode) >= 0.95
```

## Kubernetes-specific alerts

The following alerts are only focused on kubernetes and are based on metrics reported by [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics). Due to the stateful nature of this tool and the event-based nature in parts of kubernetes, these alerts are sometimes a bit hard to write and a bit wonky, but they can still provide useful insights.

To signal, that a pod is `down` (from a customer's perspective), we first check, that a pod is still running, because we don't want to alert on pods getting rotated out during a deployment. Then, we combine this using `and` on the `pod` label with the `up` metric of targets, which do have a `kubernetes_container_name`, hence are pods.

In order to be able to combine this on the `pod` metric, we need to replace the `kubernetes_pod_name` label to `pod`. This is a nice method to combine two metrics of different sources on a common attribute.

And if the pod is `running`, but not `up` (i.e. reachable), then we trigger an alert:

```yml
expr: kube_pod_container_status_running == 1 
    and on(pod) label_replace(
        up{kubernetes_container_name=~".+"}, "pod", "$1", "kubernetes_pod_name", "(.*)"
    ) == 0
```

To signal, that one, or many pods of a type are unreachable, we test if the existing replicas of a kubernetes `deployment`, are smaller than the amount of expected replicas:

```yml
expr: kube_deployment_status_replicas > kube_deployment_status_replicas_available
```

To signal, that all pods of a type are unreachable, we basically do the same as above, but we test, that no replicas are actually available, which means that the service can not be reached:

```yml
expr: kube_deployment_status_replicas > 0 
    and kube_deployment_status_replicas_available == 0
```

To signal, that a pod was restarted, we check only pods, that have been terminated and we calculate the rate of restarts during the last 5 minutes, to notice, even if the pod was restarted between prometheus polls, that it happened:

```yml
expr: kube_pod_container_status_last_terminated_reason == 1 
    and on(container) rate(kube_pod_container_status_restarts_total[5m]) * 300 > 1
```

To signal, that a pod is likely having an issue to start up, we check if a pod is in `waiting` state, but not with the reason `ContainerCreating`, which would just mean that it's starting up:

```yml
expr: kube_pod_container_status_waiting_reason{reason!="ContainerCreating"} == 1
```


## Error Rate / Response Time Alerts

The following two alerts are based on a custom counter, counting http response status codes (`gateway_status_codes`) and a summary of http response times (`gateway_response_time`). It doesn't really matter where these values come from (e.g. from querying ELK stack logs), but if they are structured in this way, they can be used for alerts.

To signal an increase in 5xx errors, we simply use the `increase` function on the counter and compare it with a threshold over a given amount of time (1m in this case). This could also be done with `4xx` errors. In our case, all 5xx errors are categorized with the `type` 500:

```yml
expr: increase(gateway_status_code{type="500"}[1m]) > 50 
```

To signal a spike in response-time, we can use the `quantile` label within the created prometheus summary and compare the quantiles with fixed values. The actual values and limits for alerting will greatly depend on the actual healthy response-times of the system:

```yml
expr: gateway_response_time{quantile="0.5"} > 0.1 
    or gateway_response_time{quantile="0.9"} > 0.5 
    or gateway_response_time{quantile="0.99"} > 1.0
```

## Reachability Alert

To finish this list up, here is a way to test if a certain url is reachable.

This alert relies on [Blackbox-Exporter](https://github.com/prometheus/blackbox_exporter) to work. I won't go into any detail on the blackbox-exporter, but suffice to say that you basically pass a list of URLs to check to it, which are tested and then you can query if these probes were successful.

So to check, if a url could not be reached, you could use the following:

```yml
expr: probe_success == 0 
```

## Conclusion

I hope this small collection of prometheus alerting examples was useful to you, or at least helped you write, or improve your own alerts. :)

#### Resources

* [prometheus](https://prometheus.io/)
* [alertmanager](https://github.com/prometheus/alertmanager)
* [node-exporter](https://github.com/prometheus/node_exporter)
* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
* [blackbox-exporter](https://github.com/prometheus/blackbox_exporter)
