# Kubernetes Metrics Server

A useful tool to deploy on the master node is the [metrics-server](https://github.com/kubernetes-sigs/metrics-server). It scans the nodes and the pods in a cluster and generates useful metrics.  The results can be seen on the command line or the dashboard.
## Get deployment manifest
From a login on the master node get the metrics server deployment yaml file 
```
[jkozik@kmaster ~]$ curl -L https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -o components.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   156  100   156    0     0    375      0 --:--:-- --:--:-- --:--:--   375
100   621  100   621    0     0   1381      0 --:--:-- --:--:-- --:--:--  1381
100  4115  100  4115    0     0   7400      0 --:--:-- --:--:-- --:--:--  7400
