# Setup Prometheus
As part of seting up the kubernetes cluster, it is useful to setup the promethes monitoring services.  There's lots of blog articles on how to install and setup.  What I found is that the procedure has changed/improved; one needs to look for the most recent procedures.

Get the values for the helm chart
```
[jkozik@dell2 ~]$ helm inspect values bitnami/kube-prometheus > prometheus.values2
[jkozik@dell2 ~]$ vi prometheus.values
```
In the temporary file prometheus.values, search for "Prometheus Service" look for a type: ClusterIP. Change it to NodePort.  This lets prometheus be exposed to a web browser on my home LAN outside of the cluster.
The file will be referenced on the install, see below.

```
[jkozik@dell2 ~]$ kubectl create namespace prometheus
namespace/prometheus created

[jkozik@dell2 ~]$ helm install prometheus bitnami/kube-prometheus --values prometheus.values --namespace prometheus
W0825 12:25:35.413893   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.417202   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.418640   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.419874   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.421232   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.811578   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.860157   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.860158   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.860157   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0825 12:25:35.860172   11825 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
NAME: prometheus
LAST DEPLOYED: Wed Aug 25 12:25:34 2021
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Watch the Prometheus Operator Deployment status using the command:

    kubectl get deploy -w --namespace prometheus -l app.kubernetes.io/name=kube-prometheus-operator,app.kubernetes.io/instance=prometheus

Watch the Prometheus StatefulSet status using the command:

    kubectl get sts -w --namespace prometheus -l app.kubernetes.io/name=kube-prometheus-prometheus,app.kubernetes.io/instance=prometheus

Prometheus can be accessed via port "9090" on the following DNS name from within your cluster:

    prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local

To access Prometheus from outside the cluster execute the following commands:

    export NODE_PORT=$(kubectl get --namespace prometheus -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-kube-prometheus-prometheus)
    export NODE_IP=$(kubectl get nodes --namespace prometheus -o jsonpath="{.items[0].status.addresses[0].address}")
    echo "Prometheus URL: http://$NODE_IP:$NODE_PORT/"

Watch the Alertmanager StatefulSet status using the command:

    kubectl get sts -w --namespace prometheus -l app.kubernetes.io/name=kube-prometheus-alertmanager,app.kubernetes.io/instance=prometheus

Alertmanager can be accessed via port "9093" on the following DNS name from within your cluster:

    prometheus-kube-prometheus-alertmanager.prometheus.svc.cluster.local

To access Alertmanager from outside the cluster execute the following commands:

    echo "Alertmanager URL: http://127.0.0.1:9093/"
    kubectl port-forward --namespace prometheus svc/prometheus-kube-prometheus-alertmanager 9093:9093
```    
It will take some time for the resources to come up.  By putting everything in one namespace, it is easy to check.  See below:
```
[jkozik@dell2 ~]$ kubectl get all --namespace prometheus
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          19m
pod/prometheus-kube-prometheus-operator-7f45dd6d9b-mfr9x     1/1     Running   0          20m
pod/prometheus-kube-state-metrics-86b9d86ccd-kjqw4           1/1     Running   0          20m
pod/prometheus-node-exporter-9zs4t                           1/1     Running   0          20m
pod/prometheus-node-exporter-lk728                           1/1     Running   0          20m
pod/prometheus-node-exporter-vkzrm                           1/1     Running   0          20m
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          19m

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   19m
service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.104.114.17    <none>        9093/TCP                     20m
service/prometheus-kube-prometheus-operator       ClusterIP   10.111.155.45    <none>        8080/TCP                     20m
service/prometheus-kube-prometheus-prometheus     NodePort    10.100.242.165   <none>        9090:30245/TCP               20m
service/prometheus-kube-state-metrics             ClusterIP   10.103.177.255   <none>        8080/TCP                     20m
service/prometheus-node-exporter                  ClusterIP   10.97.199.99     <none>        9100/TCP                     20m
service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     19m

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   3         3         3       3            3           <none>          20m

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           20m
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           20m

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-kube-prometheus-operator-7f45dd6d9b   1         1         1       20m
replicaset.apps/prometheus-kube-state-metrics-86b9d86ccd         1         1         1       20m

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     19m
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     19m
```

## Verify 
Note that the prometheus-kube-prometheus-prometheus service is of type NodePort. That means one can access the service from the local LAN using the NodePort address listed above.

In this case: http://192.168.100.172:30245   The 172 IP address is the one for the controller node for my cluster. Once at the Prometheus web page, click on Graph.   Then click on the circle button just to the left of the execute button.  Scroll down to the paramenter called node_load15.  Select it, then press the Execute button. The results will show as both a table and a graph.  Select the graph tab.

# Setup Grafana
The references bundle Grafana with Prometheus.  Here's what I did to install it.

First get the values file from the helm chart repository:
```
[jkozik@dell2 ~]$ helm inspect values bitnami/grafana > granfana.values

[jkozik@dell2 ~]$ vi grafana.values
... inside the file, I edited a couple of lines
  storageClass: "nfs-client"
    password: "pickapassword"
```
Then I installed the helm chart into a grafana namespace:
```
[jkozik@dell2 ~]$ kubectl create namespace grafana
[jkozik@dell2 ~]$ helm install  grafana bitnami/grafana --values grafana.values --namespace grafana
NAME: grafana
LAST DEPLOYED: Thu Aug 26 15:36:52 2021
NAMESPACE: grafana
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

1. Get the application URL by running these commands:
    export NODE_PORT=$(kubectl get --namespace grafana -o jsonpath="{.spec.ports[0].nodePort}" services grafana)
    export NODE_IP=$(kubectl get nodes --namespace grafana -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT

2. Get the admin credentials:

    echo "User: admin"
    echo "Password: $(kubectl get secret grafana-admin --namespace grafana -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode)"
```
## Verify Grafana is running
Then I quickly verified that everything was running. 
```
[jkozik@dell2 ~]$ kubectl get all,pv,pvc  --namespace grafana
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-6d788dcb84-25plm   0/1     Running   0          71s

NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/grafana   NodePort   10.109.117.30   <none>        3000:32174/TCP   71s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   0/1     1            0           71s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-6d788dcb84   1         1         0       71s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS   REASON   AGE
persistentvolume/chwcom-persistent-storage                  1Gi        ROX            Retain           Bound    default/chwcom-persistent-storage      nfs                     44d
persistentvolume/mysql-persistent-storage                   1Gi        RWX            Retain           Bound    default/mysql-persistent-storage                               63d
persistentvolume/nfs-pv                                     1Gi        RWX            Retain           Bound    default/nfs-pvc                        nfs                     65d
persistentvolume/nginx-persistent-storage                   1Gi        RWX            Retain           Bound    default/nginx-persistent-storage                               63d
persistentvolume/nwcom-persistent-storage                   1Gi        ROX            Retain           Bound    default/nwcom-persistent-storage       nfs                     40d
persistentvolume/pvc-a13d1699-0a2f-422d-bdcf-c63246bb2ee4   10Gi       RWO            Delete           Bound    grafana/grafana                        nfs-client              71s
persistentvolume/wordpress-persistent-storage               1Gi        RWX            Retain           Bound    default/wordpress-persistent-storage                           63d

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/grafana   Bound    pvc-a13d1699-0a2f-422d-bdcf-c63246bb2ee4   10Gi       RWO            nfs-client     71s
```
The Grafana portal is reachable from http://192.168.100.172:32174. 

## Configure Grafana

Login with the default user id and password (you can set those in the grafana.values file, above).  It will prompt you to change your password. 

Once in Grafana, create a linkage to prometheus: Find the settings icon and select Data Sources.  Click on add data source, find Prometheus and click on select.  In the http field type the URL to the Prometheus server, select browser and click on Save and Test.  This should work.  For me, I got errors; using the browser debugger window I discovered that I got CORS security errors.  I needed to use nginx to fix this.  If this happens see my next section on Ingress Setup.  

With a Data Source, create a simple data board to verify operations.  Click on the + sign, create dashboard. Click Add panel, add empty panel. Find the Metrics browser > line and enter "Node_load15", click on the link that pops up.  After a short while, an aggregate chart of the performance of all the nodes in the cluster will display on the graph above.  Title the panel "Node_load15" apply and then title the Dashboard as "Node_load15" and save it. 

So this basically works, but it is boring.  The Grafana value comes from the library of dashboards supplied by the community.  Look for the article titled [10 Most Useful Grafana Dashboards to Monitor Kubernetes and Services](https://blog.lwolf.org/post/going-open-source-in-monitoring-part-iii-10-most-useful-grafana-dashboards-to-monitor-kubernetes-and-services/). For example pick dashboard 1621.  

From the home screen, click on + Import. In the field "Import via grafana.com" enter 1621; press Load. In the next dialog box select a Prometheus Data source. And click on Import. Wait awhile, some of the panels will take longer than others.  From here it is a matter of personal taste.  

# Setup Ingress

Both Prometheus and Grafana are exposed as NodePort services.  To connect them to real URLs, create Ingress rules.  I already setup an Ingress Controller for my applications See [kubernetes Ingress Controller setup and testing](https://github.com/jkozik/k8sIngressTest).  The ingress rules will take advantage of it. The following manifests mapped the services to http://prometheusk8s.kozik.net and http://grafanak8s.kozik.net.   
## Create manifest
```
cat <<EOF >>nginx-wordpress-ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  namespace: prometheus
spec:
  rules:
  - host: prometheusk8s.kozik.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  namespace: grafana
spec:
  rules:
  - host: grafanak8s.kozik.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000       
EOF 
## Apply ingress manifest for prometheus and grafana
[jkozik@dell2 ~]$ kubectl apply -f prometheus-grafana-ingress.yaml
ingress.networking.k8s.io/prometheus unchanged
ingress.networking.k8s.io/grafana created

## Verify

[jkozik@dell2 ~]$ kubectl get all,ing -n grafana
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-6d788dcb84-25plm   1/1     Running   0          21h

NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/grafana   NodePort   10.109.117.30   <none>        3000:32174/TCP   21h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           21h

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-6d788dcb84   1         1         1       21h

NAME                                CLASS    HOSTS                  ADDRESS           PORTS   AGE
ingress.networking.k8s.io/grafana   <none>   grafanak8s.kozik.net   192.168.100.174   80      50s

[jkozik@dell2 ~]$ kubectl get all,ing -n prometheus
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          2d
pod/prometheus-kube-prometheus-operator-7f45dd6d9b-mfr9x     1/1     Running   0          2d
pod/prometheus-kube-state-metrics-86b9d86ccd-kjqw4           1/1     Running   0          2d
pod/prometheus-node-exporter-9zs4t                           1/1     Running   0          2d
pod/prometheus-node-exporter-lk728                           1/1     Running   0          2d
pod/prometheus-node-exporter-vkzrm                           1/1     Running   0          2d
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          2d

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2d
service/prometheus-kube-prometheus-alertmanager   ClusterIP   10.104.114.17    <none>        9093/TCP                     2d
service/prometheus-kube-prometheus-operator       ClusterIP   10.111.155.45    <none>        8080/TCP                     2d
service/prometheus-kube-prometheus-prometheus     NodePort    10.100.242.165   <none>        9090:30245/TCP               2d
service/prometheus-kube-state-metrics             ClusterIP   10.103.177.255   <none>        8080/TCP                     2d
service/prometheus-node-exporter                  ClusterIP   10.97.199.99     <none>        9100/TCP                     2d
service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     2d

NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-node-exporter   3         3         3       3            3           <none>          2d

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           2d
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           2d

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-kube-prometheus-operator-7f45dd6d9b   1         1         1       2d
replicaset.apps/prometheus-kube-state-metrics-86b9d86ccd         1         1         1       2d

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     2d
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     2d

NAME                                   CLASS    HOSTS           ADDRESS           PORTS   AGE
ingress.networking.k8s.io/prometheus   <none>   k8s.kozik.net   192.168.100.174   80      19s
```

# Apache Exporter
Out of the box, Prometheus and Grafana are wired to monitor the performance of a Kubernetes cluster. But Prometheus is designed for application monitoring.  There's a whole [library of Prometheus exporters](https://prometheus.io/docs/instrumenting/exporters/) that give visibility to the performance of an application. I wanted to test this out, so here's my notes on the Apache Exporter. 

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

This sidecar container pulls data from the server-status and formats it into a prometheus metrics page at the prometheus standards port 9117. So my container is technicall running two http endpoints, one is the apache httpd used by my application and the other is a go-based httpd server on port 9117 that contains a reformating of server-status data.   See the crude example below:
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
A useful tutorial: [Prometheus: Apache Exporter – Install and Config – Ubuntu, CentOS](https://www.shellhacks.com/prometheus-apache-exporter-install-config-ubuntu-centos/) and see also [bitnami/bitnami-docker-apache-exporter](https://github.com/bitnami/bitnami-docker-apache-exporter). 

## Still to do -- link my application to Prometheus


# References
- https://www.youtube.com/watch?v=CmPdyvgmw-A&t=937s
- https://www.google.com/search?q=prometheus+for+kubernetes+install&rlz=1C1GCEU_enUS883US883&oq=prometheus+for+kubernetes+install&aqs=chrome..69i57.7222j0j7&sourceid=chrome&ie=UTF-8
- https://github.com/prometheus-operator/kube-prometheus
- https://computingforgeeks.com/setup-prometheus-and-grafana-on-kubernetes/
- https://computingforgeeks.com/install-and-use-helm-3-on-kubernetes-cluster/
- https://www.magalix.com/blog/monitoring-of-kubernetes-cluster-through-prometheus-and-grafana
- https://tanzu.vmware.com/developer/guides/kubernetes/observability-prometheus-grafana-p1/
- https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/
- 
