---
layout: post
title:  "Grafana Alloy/Mimir - Received a series with duplicate label name"
author: Ivan
date:   2025-01-30 10:37:00 +0200
categories: Grafana Alloy Mimir Prometheus Kubernetes
---

## TL/DR
Just sure to restart your Alloy instances from time to time

## A Monitoring tale
I've been setting up a [LG(T)M](https://grafana.com/go/webinar/getting-started-with-grafana-lgtm-stack/) stack to monitor a kubernetes cluster of mine. It's a self-built setup running on [Hetzner Cloud](https://www.hetzner.com/cloud) using the excellent [Kube-Hetzner](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner) project to configure the cluster itself. I'll probably do a write-up on that setup at a later point, suffice to say that it's really hassle-free and works great. 

I have a monitoring/alerting set-up consisting of Grafana itself for visualisation, [Grafana Loki](https://grafana.com/oss/loki/) for log storage, [Grafana Mimir](https://grafana.com/oss/mimir/) for metrics storage and [Grafana Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/) to scrape logs and metrics. It's been running for a few months, delivering logs and metrics as requested - not much as it's a dev/demo system that does not see much usage.

Lately I've looked into pushing more workloads to the cluster and decided I need more metrics at pod level. Lookin at what was scraped, I identified two sets of metrics that I wanted in - specifically [Kube-State-Metrics](https://github.com/kubernetes/kube-state-metrics) and [cAdvisor](https://github.com/google/cadvisor/). It's a bit of a mess to set that up in **Alloy** (another piece to describe in a future post) and it involved lots of edit to the Alloy configuration file. At one point in time I started receiving some errors in both **Mimir** and **Alloy** logs:

```
ts=2025-01-29T14:12:10.555161198Z level=error msg="non-recoverable error" component_path=/ 
component_id=prometheus.remote_write.default_mimir subcomponent=rw remote_name=711a87 
url=http://mimir-nginx.monitoring.svc.cluster.local:80/api/v1/push count=707 exemplarCount=0 
err="server returned HTTP status 400 Bad Request: received a series with duplicate label name, 
label: 'node_kubernetes_io_instance_type' series: 'machine_scrape_error{beta_kubernetes_io_arch=\"amd64\",
beta_kubernetes_io_instance_type=\"cx42\", beta_kubernetes_io_os=\"linux\", csi_hetzner_cloud_location=\"hel1\",
 failure_domain_beta_kubernetes_io_regio' (err-mimir-duplicate-label-names)"
```

This is just a sample, I was getting lots of messages for different duplicate labels, not only **node_kubernetes_io_instance_type**.

Google showed up some results like [This](https://github.com/grafana/mimir/discussions/8344) and [This](https://github.com/grafana/alloy/issues/1006). However, the *Quick Workaround* mentioned didn't work for me... I dug a bit deeper and these two issues didn't really seem to be the same as mine - in both cases an **err-mimir-duplicate-label-names** was generated, but I was not doing any relabelling on my metrics at all. 

After spending a few hours fighting with this, including disabling almost all metrics collection and still getting errors I was out of ideas. 

Now a step back. I'm using Alloy in a distributed way in a *daemonset*, meaning each node gets it's own **Alloy** pod - in my case I had 6 of them. Also, I've configured **Alloy** to do live config reloads to minimize downtime. This all meant that my **Alloy** pods were up for 10+ days... In an act of desperation I followed the old Level-1 Tech support advice: **"Have you tried turning it off and on again?"**. In my case:

```shell
kubectl rollout restart daemonset.apps/alloy -n monitoring
```

When the restart was complete and the Alloy WAL was flushed from previous data, there were no more errors!

What has probably happened is that during all the configuration file restarts, Alloy instances got out of sync or just had their internal state messed up. This will probably not happen during normal operation as either:
- Config file(s) are rarely changed
- Pods get restarted from time to time
- Live reload is not used

But it's good practice to keep in mind that sometimes a good old restart is the proper solution.

## Addendum: two days later
After some more experiments I have a pretty reliable way of triggering this. Configure **Alloy** to scrape more metrics than **Mimir** will accept, you'll soon get a *HTTP 429* like so:
```
ts=2025-01-31T21:11:59.769007025Z level=warn msg="Failed to send batch, retrying" component_path=/ 
component_id=prometheus.remote_write.default_mimir subcomponent=rw remote_name=183919 
url=http://mimir-nginx.monitoring.svc.cluster.local:80/api/v1/push 
err="server returned HTTP status 429 Too Many Requests: the request has been rejected 
because the tenant exceeded the ingestion rate limit, set to 10000 items/s with a maximum 
allowed burst of 200000. This limit is applied on the total number of samples, exemplars 
and metadata received across all distributors (err-mimir-tenant-max-ingestion-rate). 
To adjust the related per-tenant limits, configure -distributor.ingestion-rate-limit 
and -distributor.ingestion-burst-size, or contact your service administrator."
```

Fair enough, Adjust the limits or reduce metrics volume and the error should go away, right? Yep, but more often than not **err-mimir-duplicate-label-names** will start popping up, prompting a restart of **Alloy**.

## Bonus: Singleline query Kubernetes API from within a pod with curl
For my own easy reference:
```shell
curl \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt   \
  --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc.cluster.local/api/v1/nodes/${NODE}/proxy/pods
```