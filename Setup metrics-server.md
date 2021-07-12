# Kubernetes Metrics Server

A useful tool to deploy on the master node is the [metrics-server](https://github.com/kubernetes-sigs/metrics-server). It scans the nodes and the pods in a cluster and generates useful metrics.  The results can be seen on the command line or the dashboard. The key new commands enabled are kubectl top nodes and Kubectl top pods. See below for example output. 
## Get deployment manifest
From a login on the master node get the metrics server deployment yaml file 
```
[jkozik@kmaster ~]$ curl -L https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -o components.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   156  100   156    0     0    375      0 --:--:-- --:--:-- --:--:--   375
100   621  100   621    0     0   1381      0 --:--:-- --:--:-- --:--:--  1381
100  4115  100  4115    0     0   7400      0 --:--:-- --:--:-- --:--:--  7400

[jkozik@kmaster ~]$ mv components.yaml metrics.yaml
```
The yaml file won't quite work, one of the parameters needs to be fixed.  Edit the file as follows:
```
vi metrics.html
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls    <-- Add this line.  The README.md in the git for metrics server talks about this.
        image: k8s.gcr.io/metrics-server/metrics-server:v0.5.0
```        
## Deploy metrics-server, verify that it is running
```
[jkozik@kmaster ~]$ kubectl apply -f metrics.yaml
serviceaccount/metrics-server unchanged
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/system:metrics-server unchanged
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader unchanged
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator unchanged
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server unchanged
service/metrics-server unchanged
deployment.apps/metrics-server configured
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io unchanged

[jkozik@kmaster ~]$ kubectl get pods -n kube-system -o wide
NAME                                       READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
calico-kube-controllers-78d6f96c7b-kxhtv   1/1     Running   0          39d     10.68.189.3       kmaster    <none>           <none>
calico-node-6x85w                          1/1     Running   0          38d     192.168.100.173   kworker1   <none>           <none>
calico-node-ch2bs                          1/1     Running   0          38d     192.168.100.172   kmaster    <none>           <none>
calico-node-zdmmt                          1/1     Running   16         38d     192.168.100.174   kworker2   <none>           <none>
coredns-558bd4d5db-dtdzd                   1/1     Running   0          39d     10.68.189.2       kmaster    <none>           <none>
coredns-558bd4d5db-rmfhz                   1/1     Running   0          39d     10.68.189.1       kmaster    <none>           <none>
etcd-kmaster                               1/1     Running   0          39d     192.168.100.172   kmaster    <none>           <none>
kube-apiserver-kmaster                     1/1     Running   0          39d     192.168.100.172   kmaster    <none>           <none>
kube-controller-manager-kmaster            1/1     Running   0          39d     192.168.100.172   kmaster    <none>           <none>
kube-proxy-2j6s7                           1/1     Running   0          38d     192.168.100.174   kworker2   <none>           <none>
kube-proxy-lg5qd                           1/1     Running   0          39d     192.168.100.172   kmaster    <none>           <none>
kube-proxy-xm4pt                           1/1     Running   0          39d     192.168.100.173   kworker1   <none>           <none>
kube-scheduler-kmaster                     1/1     Running   0          39d     192.168.100.172   kmaster    <none>           <none>
metrics-server-5cd859f5c-4gczq             0/1     Running   0          25s     10.68.77.144      kworker2   <none>           <none>

[jkozik@kmaster ~]$ kubectl logs -n kube-system metrics-server-5cd859f5c-4gczq
I0712 18:37:33.812792       1 serving.go:341] Generated self-signed cert (/tmp/apiserver.crt, /tmp/apiserver.key)
I0712 18:37:34.344867       1 secure_serving.go:197] Serving securely on [::]:443
I0712 18:37:34.344959       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
I0712 18:37:34.344967       1 shared_informer.go:240] Waiting for caches to sync for RequestHeaderAuthRequestController
I0712 18:37:34.344982       1 dynamic_serving_content.go:130] Starting serving-cert::/tmp/apiserver.crt::/tmp/apiserver.key
I0712 18:37:34.345007       1 tlsconfig.go:240] Starting DynamicServingCertificateController
I0712 18:37:34.345763       1 configmap_cafile_content.go:202] Starting client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0712 18:37:34.345773       1 shared_informer.go:240] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0712 18:37:34.345782       1 configmap_cafile_content.go:202] Starting client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0712 18:37:34.345785       1 shared_informer.go:240] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0712 18:37:34.445816       1 shared_informer.go:247] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0712 18:37:34.445853       1 shared_informer.go:247] Caches are synced for RequestHeaderAuthRequestController
I0712 18:37:34.445928       1 shared_informer.go:247] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file

[jkozik@kmaster ~]$ kubectl top nodes
W0712 13:38:36.767824   24518 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kmaster    324m         16%    1304Mi          75%
kworker1   105m         10%    1186Mi          68%
kworker2   118m         11%    599Mi           67%

[jkozik@kmaster ~]$ kubectl top pods
W0712 14:40:45.269370   20007 top_pod.go:140] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
NAME                                   CPU(cores)   MEMORY(bytes)
apple-app                              0m           1Mi
banana-app                             1m           1Mi
chwcom-6f48d9b7cc-jrc9j                1m           57Mi
kubernetes-bootcamp-57978f5f5d-6zhx6   0m           8Mi
nginx-deployment-56fcf9d48b-nf8kd      0m           2Mi
wordpress-6db6fb5444-5kd7p             1m           66Mi
wordpress-mysql-66cb796566-bbqdg       1m           459Mi
```

Note:  after I applied the metrics server yaml file, it took a few minutes before the pod was ready and the log files were clean.  Also, these same metrics display graphically on the kubernetes dashboard.  Worth it.

This is useful for me because I am trying to keep my cluster size small and I want to monitor how big my pods are getting.

## References
- https://www.youtube.com/watch?v=unhB90kwc4Q
- https://www.youtube.com/watch?v=PEs2ccoZ3ow
- https://github.com/kubernetes-sigs/metrics-server
