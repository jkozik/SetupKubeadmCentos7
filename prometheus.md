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
