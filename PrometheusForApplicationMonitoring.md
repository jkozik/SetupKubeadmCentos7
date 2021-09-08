# Prometheus monitoring of applications

Earlier, I installed a prometheus-operator for monitoring the kubernetes cluster.  [See my notes.](https://github.com/jkozik/SetupKubeadmCentos7/blob/main/prometheus.md). The helm chart installed a big stack including the ServiceMonitors required to monitor the kubernetes cluster.  

In this note, I will show my steps for taking the previously installed prometheus operator and configure it to monitor my deployments.  The initial case are the various weather websites that I run including SanCapWeather.com.  This is basically an apache web site.  Prometheus has apache exporters designed to work with applications like this. 

Here's some articles on this subject: 
- [Kubernetes prometheus operator deployment](https://ervikrant06.github.io/kubernetes/Kuberenetes-prometheus-installation/)
- [Monitoring Kubernetes with prometheus-operator - Lili Cosic, Red Hat](https://www.youtube.com/watch?v=MuHPMXCGiLc).  I especially refered to it starting at 19:05
- [Introduction to the Prometheus Operator on Kubernetes](https://www.youtube.com/watch?v=LQpmeb7idt8) which uses [this github repository](https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/monitoring/prometheus/kubernetes/1.14.8/prometheus-standalone) .  The prometheus stand-alone discussion starts at 10:30. 
- [Kubernetes Cluster Monitoring for beginners](https://www.youtube.com/watch?v=TrGgvkJfslw)

## Example test application
To start, I want to setup a very simple application and verify that prometheus can discover it.  

In the github repository for [prometheus-operator](https://github.com/prometheus-operator) there is an [example application directory](https://github.com/prometheus-operator/kube-prometheus/tree/main/examples/example-app) that creates a dummy application service/deployment/pod and applies a ServiceMonitor to link the application to the prometheus operator previously installed.

I cut and pasted the key files into one here document:

```
cat <<EOF >> prometheus-example-app.yaml
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    tier: frontend
  namespace: default
spec:
  selector:
    app: example-app
  ports:
  - name: web
    protocol: TCP
    port: 8080
    targetPort: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: example-app
      version: 1.1.3
  replicas: 4
  template:
    metadata:
      labels:
        app: example-app
        version: 1.1.3
    spec:
      containers:
      - name: example-app
        image: quay.io/fabxc/prometheus_demo_service
        ports:
        - name: web
          containerPort: 8080
          protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: frontend
  namespace: default
  labels:
    tier: frontend
spec:
  selector:
    matchLabels:
      tier: frontend
  targetLabels:
    - tier
  endpoints:
  - port: web
    interval: 10s
  namespaceSelector:
    matchNames:
      - default
EOF
```
Some notes about the above resources:  look at the deployment, 'name: example-app', it has the label 'app: example-app'. It is exposed via the service 'name: example-app' which has a selector 'app: example-app' and has a label 'tier: frontend'.  The service exposes all deployments with a label of 'app: exmaple-app'.

The ServiceMonitor named "name: frontend", it has a selector: 'matchLabels: tieir: frontend'. It tells prometheus to scrape all resources attached to a service that matches the label 'tier: frontend'.  

This linkage is very generalizable.  Among other things, it envisions deployments to be designed to bundle together a ServiceMonitor resource along with the other typical resources that make up an application in kubernetes:  deployements, services, ingress, pv, pvc, ... and servicemonitor.

## Deploy example-app

```
[jkozik@dell2 ~]$ kubectl apply -f prometheus-example-app.yaml
service/example-app unchanged
deployment.apps/example-app unchanged
servicemonitor.monitoring.coreos.com/frontend unchanged

[jkozik@dell2 ~]$ kubectl get servicemonitor frontend
NAME       AGE
frontend   4d19h

[jkozik@dell2 ~]$ kubectl get svc,deploy example-app -owide
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/example-app   ClusterIP   10.107.193.117   <none>        8080/TCP   4d19h   app=example-app

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS    IMAGES                                  SELECTOR
deployment.apps/example-app   4/4     4            4           4d19h   example-app   quay.io/fabxc/prometheus_demo_service   app=example-app,version=1.1.3
[jkozik@dell2 ~]$

```

## From prometheus portal, verify the target exists, and metrics are being generated

From here, go to the prometheus portal page, click on status->targets.  Look for the ServiceMonitor in the default name space called front end (serviceMonitor/default/frontend/).  Verify that you see the example-app running. In the configuration above, 4 replicas were defined. One should see 4 target IP addresses.

The endpoints look like http://10.68.140.21:8080/metrics. This address can only be accessed from inside the cluster.  Veryify that the metrics look sane.  Log on to one of the nodes of the cluster and try to fetch metrics data.  See below:

```
[jkozik@kmaster ~]$ curl http://10.68.140.21:8080/metrics
# HELP demo_api_http_requests_in_progress The current number of API HTTP requests in progress.
# TYPE demo_api_http_requests_in_progress gauge
demo_api_http_requests_in_progress 3
# HELP demo_api_request_duration_seconds A histogram of the API HTTP request durations in seconds.
# TYPE demo_api_request_duration_seconds histogram
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.0001"} 0
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.00015000000000000001"} 0
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.00022500000000000002"} 0
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.0003375"} 0
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.00050625"} 0
demo_api_request_duration_seconds_bucket{method="GET",path="/api/bar",status="200",le="0.000759375"} 0
...
```
It was about 2 screens worth of metrics, I only show the first few lines.  This means things are basically working.

Next in the prometheus portal, click on graphes.  In the search bar type demo... you should see a bunch of metrics from the example-app list.  Type in demo_api_http_requests_in_progress and click on execute.  You should see an interesting graph.

Finally, query all the parameters from the example application: sum({job="example-app"}) by(__name__).  Using contrl-click you can select multiple metrics and have them graphed together.


# Apache Exporter
Out of the box, Prometheus and Grafana are wired to monitor the performance of a Kubernetes cluster. But Prometheus is designed for application monitoring.  There's a whole [library of Prometheus exporters](https://prometheus.io/docs/instrumenting/exporters/) that give visibility to the performance of an application. I wanted to test this out, so here's my notes on the Apache Exporter. 

## Verify that server-status works
I have an application called sancapweather.com (app=scwcom).  It runs my weather website; it is based on Apache web server.  It has built into in the [mod-status](https://httpd.apache.org/docs/2.4/mod/mod_status.html)-based tool called server-status.  I have this turned-on; this is a normal part of apache website configurations, out side the scope of this writeup.  Here's a crude demonstration running inside the scwcom pod:
```
[jkozik@dell2 ~]$ kubectl get pod --selector=app=scwcom
NAME                           READY   STATUS      RESTARTS   AGE
saveyesterday-27173758-f27mc   0/1     Completed   0          131m
saveyesterday-27173818-cr9kl   0/1     Completed   0          71m
saveyesterday-27173878-dhhw9   0/1     Completed   0          11m
scwcom-5f647c54cc-b5hfq        2/2     Running     0          24h

[jkozik@dell2 ~]$ kubectl exec -it scwcom-5f647c54cc-b5hfq -- /bin/bash
Defaulting container name to scwcom.
Use 'kubectl describe pod/scwcom-5f647c54cc-b5hfq -n default' to see all of the containers in this pod.
root@scwcom-5f647c54cc-b5hfq:/var/www/html# curl http://127.0.0.1/server-status
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html><head>
<title>Apache Status</title>
</head><body>
<h1>Apache Server Status for 127.0.0.1 (via 127.0.0.1)</h1>

<dl><dt>Server Version: Apache/2.4.38 (Debian) PHP/7.2.34</dt>
<dt>Server MPM: prefork</dt>
<dt>Server Built: 2020-08-25T20:08:29
</dt></dl><hr /><dl>
<dt>Current Time: Tuesday, 31-Aug-2021 18:10:01 UTC</dt>
<dt>Restart Time: Monday, 30-Aug-2021 17:14:41 UTC</dt>
<dt>Parent Server Config. Generation: 1</dt>
<dt>Parent Server MPM Generation: 0</dt>
<dt>Server uptime:  1 day 55 minutes 20 seconds</dt>
<dt>Server load: 0.11 0.17 0.20</dt>
```
## Attach Apache Exporter to my SanCapWeather.com (scwcom) application as a sidecar
I found the [official documentation for the Apache Exporter in github](https://github.com/Lusitaniae/apache_exporter).  A build was posted in docker hub as [lusotycoon/apache-exporter](https://hub.docker.com/r/lusotycoon/apache-exporter/).  Then following [the example from Cloud Academy](https://github.com/cloudacademy/Apache-Prometheus-Exporter), I added a second container to my scwcom pod using this image.  See example scwcom.deployment.yaml page:
```
[jkozik@dell2 k8s]$ cat scwcom-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scwcom
  labels:
    app: scwcom
    fqdn: sancapweather.com
spec:
  selector:
    matchLabels:
      app: scwcom
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: scwcom
        fqdn: sancapweather.com
    spec:
      containers:
      - image: jkozik/scw.com
        name: scwcom
        ports:
        - containerPort: 80
          name: scwcom
      - image: lusotycoon/apache-exporter
        name: apache-exporter
        ports:
        - containerPort: 9117
[jkozik@dell2 k8s]$
```
Look at the second image and note the port number.  This is following the example in the [Cloud Academy article](https://github.com/cloudacademy/Apache-Prometheus-Exporter), see the deployement.yaml file. I applied the above deployement file.

This sidecar container pulls data from the server-status and formats it into a prometheus metrics page at the prometheus standards port 9117. So my container is technically running two http endpoints, one is the apache httpd used by my application and the other is a go-based httpd server on port 9117 that contains a reformating of server-status data.   See the crude example below:
```
[jkozik@dell2 k8s]$ kubectl exec -it scwcom-5f647c54cc-b5hfq -- /bin/bash
Defaulting container name to scwcom.
Use 'kubectl describe pod/scwcom-5f647c54cc-b5hfq -n default' to see all of the containers in this pod.

root@scwcom-5f647c54cc-b5hfq:/var/www/html# curl http://localhost:9117/metrics
# HELP apache_accesses_total Current total apache accesses (*)
# TYPE apache_accesses_total counter
apache_accesses_total 96
# HELP apache_cpu_time_ms_total Apache CPU time
# TYPE apache_cpu_time_ms_total counter
apache_cpu_time_ms_total{type="system"} 3990
apache_cpu_time_ms_total{type="user"} 4130
# HELP apache_cpuload The current percentage CPU used by each worker and in total by all workers combined (*)
# TYPE apache_cpuload gauge
apache_cpuload 0.00889668
# HELP apache_duration_ms_total Total duration of all registered requests in ms
# TYPE apache_duration_ms_total counter
apache_duration_ms_total 359024
# HELP apache_exporter_build_info A metric with a constant '1' value labeled by version, revision, branch, and goversion from which apache_exporter was built.
# TYPE apache_exporter_build_info gauge
apache_exporter_build_info{branch="HEAD",goversion="go1.16.6",revision="c64b4496c4658d72c58fbda905a70c06a4b7f0c7",version="0.10.0"} 1
# HELP apache_generation Apache restart generation
# TYPE apache_generation gauge
apache_generation{type="config"} 1
apache_generation{type="mpm"} 0
# HELP apache_info Apache version information
# TYPE apache_info gauge
apache_info{mpm="prefork",version="Apache/2.4.38 (Debian) PHP/7.2.34"} 1
# HELP apache_load Apache server load
# TYPE apache_load gauge
apache_load{interval="15min"} 0.29
apache_load{interval="1min"} 0.33
apache_load{interval="5min"} 0.26
# HELP apache_scoreboard Apache scoreboard statuses
# TYPE apache_scoreboard gauge
apache_scoreboard{state="closing"} 0
apache_scoreboard{state="dns"} 0
apache_scoreboard{state="graceful_stop"} 0
apache_scoreboard{state="idle"} 9
apache_scoreboard{state="idle_cleanup"} 0
apache_scoreboard{state="keepalive"} 0
apache_scoreboard{state="logging"} 0
apache_scoreboard{state="open_slot"} 140
apache_scoreboard{state="read"} 0
apache_scoreboard{state="reply"} 1
apache_scoreboard{state="startup"} 0
# HELP apache_sent_kilobytes_total Current total kbytes sent (*)
# TYPE apache_sent_kilobytes_total counter
apache_sent_kilobytes_total 710
# HELP apache_up Could the apache server be reached
# TYPE apache_up gauge
apache_up 1
# HELP apache_uptime_seconds_total Current uptime in seconds (*)
# TYPE apache_uptime_seconds_total counter
apache_uptime_seconds_total 91270
# HELP apache_version Apache server version
# TYPE apache_version gauge
apache_version 2.04038
# HELP apache_workers Apache worker statuses
# TYPE apache_workers gauge
apache_workers{state="busy"} 1
apache_workers{state="idle"} 9
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 9
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.16.6"} 1
# HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
# TYPE go_memstats_alloc_bytes gauge
go_memstats_alloc_bytes 1.168944e+06
# HELP go_memstats_alloc_bytes_total Total number of bytes allocated, even if freed.
# TYPE go_memstats_alloc_bytes_total counter
go_memstats_alloc_bytes_total 1.168944e+06
# HELP go_memstats_buck_hash_sys_bytes Number of bytes used by the profiling bucket hash table.
# TYPE go_memstats_buck_hash_sys_bytes gauge
go_memstats_buck_hash_sys_bytes 1.444741e+06
# HELP go_memstats_frees_total Total number of frees.
# TYPE go_memstats_frees_total counter
go_memstats_frees_total 436
# HELP go_memstats_gc_cpu_fraction The fraction of this program's available CPU time used by the GC since the program started.
# TYPE go_memstats_gc_cpu_fraction gauge
go_memstats_gc_cpu_fraction 0
# HELP go_memstats_gc_sys_bytes Number of bytes used for garbage collection system metadata.
# TYPE go_memstats_gc_sys_bytes gauge
go_memstats_gc_sys_bytes 4.03616e+06
# HELP go_memstats_heap_alloc_bytes Number of heap bytes allocated and still in use.
# TYPE go_memstats_heap_alloc_bytes gauge
go_memstats_heap_alloc_bytes 1.168944e+06
# HELP go_memstats_heap_idle_bytes Number of heap bytes waiting to be used.
# TYPE go_memstats_heap_idle_bytes gauge
go_memstats_heap_idle_bytes 6.434816e+07
# HELP go_memstats_heap_inuse_bytes Number of heap bytes that are in use.
# TYPE go_memstats_heap_inuse_bytes gauge
go_memstats_heap_inuse_bytes 2.269184e+06
# HELP go_memstats_heap_objects Number of allocated objects.
# TYPE go_memstats_heap_objects gauge
go_memstats_heap_objects 4948
# HELP go_memstats_heap_released_bytes Number of heap bytes released to OS.
# TYPE go_memstats_heap_released_bytes gauge
go_memstats_heap_released_bytes 6.4299008e+07
# HELP go_memstats_heap_sys_bytes Number of heap bytes obtained from system.
# TYPE go_memstats_heap_sys_bytes gauge
go_memstats_heap_sys_bytes 6.6617344e+07
# HELP go_memstats_last_gc_time_seconds Number of seconds since 1970 of last garbage collection.
# TYPE go_memstats_last_gc_time_seconds gauge
go_memstats_last_gc_time_seconds 0
# HELP go_memstats_lookups_total Total number of pointer lookups.
# TYPE go_memstats_lookups_total counter
go_memstats_lookups_total 0
# HELP go_memstats_mallocs_total Total number of mallocs.
# TYPE go_memstats_mallocs_total counter
go_memstats_mallocs_total 5384
# HELP go_memstats_mcache_inuse_bytes Number of bytes in use by mcache structures.
# TYPE go_memstats_mcache_inuse_bytes gauge
go_memstats_mcache_inuse_bytes 3600
# HELP go_memstats_mcache_sys_bytes Number of bytes used for mcache structures obtained from system.
# TYPE go_memstats_mcache_sys_bytes gauge
go_memstats_mcache_sys_bytes 16384
# HELP go_memstats_mspan_inuse_bytes Number of bytes in use by mspan structures.
# TYPE go_memstats_mspan_inuse_bytes gauge
go_memstats_mspan_inuse_bytes 36856
# HELP go_memstats_mspan_sys_bytes Number of bytes used for mspan structures obtained from system.
# TYPE go_memstats_mspan_sys_bytes gauge
go_memstats_mspan_sys_bytes 49152
# HELP go_memstats_next_gc_bytes Number of heap bytes when next garbage collection will take place.
# TYPE go_memstats_next_gc_bytes gauge
go_memstats_next_gc_bytes 4.473924e+06
# HELP go_memstats_other_sys_bytes Number of bytes used for other system allocations.
# TYPE go_memstats_other_sys_bytes gauge
go_memstats_other_sys_bytes 697155
# HELP go_memstats_stack_inuse_bytes Number of bytes in use by the stack allocator.
# TYPE go_memstats_stack_inuse_bytes gauge
go_memstats_stack_inuse_bytes 491520
# HELP go_memstats_stack_sys_bytes Number of bytes obtained from system for stack allocator.
# TYPE go_memstats_stack_sys_bytes gauge
go_memstats_stack_sys_bytes 491520
# HELP go_memstats_sys_bytes Number of bytes obtained from system.
# TYPE go_memstats_sys_bytes gauge
go_memstats_sys_bytes 7.3352456e+07
# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
go_threads 8
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 0.11
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1.048576e+06
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 9
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 6.803456e+06
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.63034368157e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 7.30210304e+08
# HELP process_virtual_memory_max_bytes Maximum amount of virtual memory available in bytes.
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes -1
# HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.
# TYPE promhttp_metric_handler_requests_in_flight gauge
promhttp_metric_handler_requests_in_flight 1
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 1
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0
root@scwcom-5f647c54cc-b5hfq:/var/www/html#
```
Useful tutorials: 
- [Prometheus: Apache Exporter – Install and Config – Ubuntu, CentOS](https://www.shellhacks.com/prometheus-apache-exporter-install-config-ubuntu-centos/) and see also
- [bitnami/bitnami-docker-apache-exporter](https://github.com/bitnami/bitnami-docker-apache-exporter). 

## Expose the metrics port
The deployment above added a second image and added port 9117. To make all of this work, the service for scwcom needs to add an additional port.  See the updates yaml file below:
```
[jkozik@dell2 k8s]$ cat scwcom-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: scwcom
  labels:
    app: scwcom
    fqdn: sancapweather.com
spec:
  ports:
    #- port: 80
    - name: http
      port: 80
    - name: metrics
      port: 9117
  selector:
    app: scwcom
  type: NodePort
[jkozik@dell2 k8s]$
```
The key things here:  the port property is an array and it can take more than one port.  If more than one port is used, they should be named.  The port numbers should agree with the port numbers defined in the 'kubectl describe deployment scwcom'. 
```
[jkozik@dell2 k8s]$ kubectl apply -f *svc*
service/scwcom configured

[jkozik@dell2 k8s]$ kubectl get svc -o wide
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE     SELECTOR
scwcom                      NodePort       10.104.191.227   <none>        80:31436/TCP,9117:31418/TCP   51d     app=scwcom
```
Note: scwcom service now exposes two ports on the NodePort IP address. 
## Verify that the scwcom service exposes the metrics
As with any NodePort service, one can curl the cluster's ip address with the 30xxx port number.  This is an easy check from my kubectl prompt:
```
[jkozik@dell2 k8s]$ curl http://192.168.100.172:31418/metrics
# HELP apache_accesses_total Current total apache accesses (*)
# TYPE apache_accesses_total counter
apache_accesses_total 524
# HELP apache_cpu_time_ms_total Apache CPU time
# TYPE apache_cpu_time_ms_total counter
apache_cpu_time_ms_total{type="system"} 31180
apache_cpu_time_ms_total{type="user"} 25850
# HELP apache_cpuload The current percentage CPU used by each worker and in total by all workers combined (*)
# TYPE apache_cpuload gauge
apache_cpuload 0.00717271
# HELP apache_duration_ms_total Total duration of all registered requests in ms
# TYPE apache_duration_ms_total counter
apache_duration_ms_total 1.929769e+06
...
```
Note: as required, the metrics are exposed on the /metrics path. 192.168.100.172 is the kmaster node IP address. It is on my home LAN. 

## Create a ServiceMonitor for scwcom
The application scwcom exposes port 9117 (aka metrics) for prometheus to scrape.  To tell the previously installed prometheus server about the application create the ServiceMonitor below:
```
[jkozik@dell2 k8s]$ cat scwcom-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: scwcom
  namespace: default
  labels:
    app: scwcom
spec:
  selector:
    matchLabels:
      app: scwcom
  endpoints:
  - port: metrics
  namespaceSelector:
    matchNames:
      - default
[jkozik@dell2 k8s]$
```
Note: this ServiceMonitor will look for all services in the default name space that match the label 'app: scwcom'.  It will scrape metrics from the endpoint: 'port: metrics'. 

Here's how I applied the ServiceMonitor
```
[jkozik@dell2 k8s]$ kubectl apply -f scwcom-servicemonitor.yaml
servicemonitor.monitoring.coreos.com/scwcom created
[jkozik@dell2 k8s]$ kubectl get servicemonitor
NAME       AGE
frontend   5d21h
scwcom     3s
```
## From prometheus portal, verify the target exists, and metrics are being generated
From here, go to the prometheus portal page, click on status->targets.  Look for the ServiceMonitor in the default name space called scwcom (serviceMonitor/default/scwcom/).  Verify that you see the scwcom-app running.

Next in the prometheus portal, click on graphes.  In the search bar type apache... you should see a bunch of metrics from the scwcom list.  Type in apache_cpuload and click on execute.  You should see an interesting graph.

Finally, query all the parameters from the example application: sum({job="scwcom"}) by(__name__).  Using contrl-click you can select multiple metrics and have them graphed together.

To do: screen captures from Prometheus portal and Grafana dashboard 3894






