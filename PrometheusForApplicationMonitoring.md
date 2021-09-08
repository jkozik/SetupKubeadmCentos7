# Prometheus monitoring of applications

Earlier, I installed a prometheus-operator for monitoring the kubernetes cluster.  [See my notes.](https://github.com/jkozik/SetupKubeadmCentos7/blob/main/prometheus.md). The helm chart installed a big stack including the ServiceMonitors required to monitor the kubernetes cluster.  

In this note, I will show my steps for taking the previously installed prometheus operator and configure it to monitor my deployments.  The initial case are the various weather websites that I run including SanCapWeather.com.  This is basically an apache web site.  Prometheus has apache exporters designed to work with applications like this. 

Here's some articles on this subject: 
- [Kubernetes prometheus operator deployment](https://ervikrant06.github.io/kubernetes/Kuberenetes-prometheus-installation/)
- [Monitoring Kubernetes with prometheus-operator - Lili Cosic, Red Hat](https://www.youtube.com/watch?v=MuHPMXCGiLc).  I especially refered to it starting at 19:05
- [Introduction to the Prometheus Operator on Kubernetes](https://www.youtube.com/watch?v=LQpmeb7idt8) which uses [this github repository](https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/monitoring/prometheus/kubernetes/1.14.8/prometheus-standalone) .  The prometheus stand-alone discussion starts at 10:30. 

## Example test application
To start, I want to setup a very simple application and verify that prometheus can discover it.  

In the github repository for [prometheus-operator](https://github.com/prometheus-operator) there is an [example application directory](https://github.com/prometheus-operator/kube-prometheus/tree/main/examples/example-app) that creates a dummy application service/deployment/pod and applies a ServiceMonitor to link the application to the prometheus operator previously installed.

I cut and pasted the key files into one here document:

```
cat << EOF >> prometheus-example-app.yaml
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




