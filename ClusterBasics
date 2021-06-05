# Cluster Basics
Here's a collection of useful tools and tips for checking out a new cluster.
## What's running
The two commands I do all the time.
```
[jkozik@kmaster ~]$ kubectl get nodes
NAME       STATUS   ROLES                  AGE    VERSION
kmaster    Ready    control-plane,master   3d     v1.21.1
kworker1   Ready    <none>                 2d6h   v1.21.1
kworker2   Ready    <none>                 28h    v1.21.1
[jkozik@kmaster ~]$ kubectl get pods --all-namespaces -o wide
NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE     IP                NODE       NOMINATED NODE   READINESS GATES
default                kubernetes-bootcamp-57978f5f5d-6zhx6         1/1     Running   0          2d3h    10.68.41.129      kworker1   <none>           <none>
default                nginx-6799fc88d8-x5tg6                       1/1     Running   0          13m     10.68.41.134      kworker1   <none>           <none>
default                nginx-deployment-66b6c48dd5-5cb4k            1/1     Running   0          26h     10.68.77.129      kworker2   <none>           <none>
default                nginx-deployment-66b6c48dd5-66pgb            1/1     Running   0          26h     10.68.41.133      kworker1   <none>           <none>
kube-system            calico-kube-controllers-78d6f96c7b-kxhtv     1/1     Running   0          2d22h   10.68.189.3       kmaster    <none>           <none>
kube-system            calico-node-6x85w                            1/1     Running   0          2d3h    192.168.100.173   kworker1   <none>           <none>
kube-system            calico-node-ch2bs                            1/1     Running   0          2d3h    192.168.100.172   kmaster    <none>           <none>
kube-system            calico-node-zdmmt                            1/1     Running   0          28h     192.168.100.174   kworker2   <none>           <none>
kube-system            coredns-558bd4d5db-dtdzd                     1/1     Running   0          3d      10.68.189.2       kmaster    <none>           <none>
kube-system            coredns-558bd4d5db-rmfhz                     1/1     Running   0          3d      10.68.189.1       kmaster    <none>           <none>
kube-system            etcd-kmaster                                 1/1     Running   0          3d      192.168.100.172   kmaster    <none>           <none>
kube-system            kube-apiserver-kmaster                       1/1     Running   0          3d      192.168.100.172   kmaster    <none>           <none>
kube-system            kube-controller-manager-kmaster              1/1     Running   0          3d      192.168.100.172   kmaster    <none>           <none>
kube-system            kube-proxy-2j6s7                             1/1     Running   0          28h     192.168.100.174   kworker2   <none>           <none>
kube-system            kube-proxy-lg5qd                             1/1     Running   0          3d      192.168.100.172   kmaster    <none>           <none>
kube-system            kube-proxy-xm4pt                             1/1     Running   0          2d6h    192.168.100.173   kworker1   <none>           <none>
kube-system            kube-scheduler-kmaster                       1/1     Running   0          3d      192.168.100.172   kmaster    <none>           <none>
kubernetes-dashboard   dashboard-metrics-scraper-856586f554-w8n2c   1/1     Running   0          2d2h    10.68.41.131      kworker1   <none>           <none>
kubernetes-dashboard   kubernetes-dashboard-7795fd6d89-v98dn        1/1     Running   0          45h     10.68.41.132      kworker1   <none>           <none>
```
The most basic thing you can do is deploy something.  Here's an example of deploying nginx on a cluster
```
[jkozik@kmaster ~]$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
[jkozik@kmaster ~]$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
nginx                 1/1     1            1           15s
[jkozik@kmaster ~]$ kubectl describe deployment nginx
Name:                   nginx
Namespace:              default
CreationTimestamp:      Sat, 05 Jun 2021 17:25:55 -0500
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-6799fc88d8 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  30s   deployment-controller  Scaled up replica set nginx-6799fc88d8 to 1
[jkozik@kmaster ~]$ kubectl create service nodeport nginx --tcp=80:80
service/nginx created
[jkozik@kmaster ~]$ kubectl get service
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        3d
nginx        NodePort    10.98.86.142   <none>        80:31983/TCP   25s
```
Note: the deployment creates a pod that is running nginx, but it is not externall accessible.  The service of type nodeport exposes a port that is externally visible that maps to the deployemtn.
Note the following curl commands:
```
[jkozik@kmaster ~]$ curl 192.168.100.172:31983
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[jkozik@kmaster ~]$ curl 192.168.100.173:31983
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[jkozik@kmaster ~]$ curl 192.168.100.174:31983
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
An http get to any of the ip addresses in the cluster to the port 31983 will return the default nginx welcome page.
Not shown here, any browser on my home LAN can receive the same thing.
