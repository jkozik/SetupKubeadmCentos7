# Dynamic NFS Storage Provisioning

Grafana was the first application for me to trigger the need for automating the creation of Persistant Volumes.  Grafana's helm chart comes with a Persistant Volume Claim, and the administration, me, needs to setup a PV to make the installation work.  I noticed a video that addressed the problem by installing a Dynamic Storage Provisioning pod.  It looks for unbound PVC and automatically creates a PV from an off cluster NFS share. Each new PV is a directory in that share.  

On my host, I already had an NFS server setup, I just added an additional line and created a new directory.  
```
[jkozik@dell2 ~]$ sudo cat /etc/exports
[sudo] password for jkozik:
...
/home/nfs/kubedata 192.168.100.0/24(rw,sync,no_root_squash,no_all_squash,no_subtree_check,insecure)
```
Note:  in the helm install command below, the nfs.server and nfs.path come straight from the nfs configuration file /etc/exports
```
[jkozik@dell2 nfs-provisioner]$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
"nfs-subdir-external-provisioner" has been added to your repositories

[jkozik@dell2 nfs-provisioner]$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=192.168.101.152 --set nfs.path=/home/nfs/kubedata
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Thu Aug 26 15:04:35 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None

[jkozik@dell2 nfs-provisioner]$ helm list
NAME                            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                   APP VERSION
nfs-subdir-external-provisioner default         1               2021-08-26 15:04:35.851301848 -0500 CDT deployed        nfs-subdir-external-provisioner-4.0.13  4.0.2
```
Also note, the application creates a new storage class called nfs-client.  For any applications that you want to install, you need to look for any PVCs and edit-in a line indicating a StorageClass of nfs-client.
```
[jkozik@dell2 nfs-provisioner]$ kubectl get storageclass
NAME         PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   2m46s
```
Here's an example pvc.yaml file
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-pv1
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

# References
- https://www.youtube.com/watch?v=DF3v2P8ENEg
- [Kubernetes NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

