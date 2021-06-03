# Setup kubeadm on Centos7
To setup Kubernetes on my home server, I need to create 3 nodes.  A master and two workers.  There are several procedures for getting this working (referenced below).  It turns out I had to look at several of them to finally get things working, so therefore, I am writing this note to help me keep things straight.

I am using VirtualBox and creating 3 VMs all based on the Centos7 minumum iso.  I install them from scratch to a static IP addresses on my home LAN (192.168.100.0/24). These VMs are setup with a bridged interface and can ping anything in my home LAN or Internet.  

Kubernetes is setup in a cluster of VMs -- each VM is called a node.  kubeadm is the tool that links the nodes. kubectl is the tool to control the cluster.  The nodes run pods which are made of docker containers. The pods network across the cluster using a tool called Calico. The steps below setup the Centos7 environment; install the main docker and kubernetes tools.  The node called kmaster is the controller node.  The two other nodes, kworker1, kworker2 run the applications in pods.  

The setup for the kmaster and the kworker1/2 nodes are the same until the cluster is created.  The kubeadm tool initializes a cluster and creates a network.  The tool generates a join script that the kworker1/2 nodes run to join the cluster. The kubectl command is used to verify that the cluster setup was completed successfully.

Most of the steps below are common to all three nodes.

## Setup kmaster, kworker1 and kworker2
In root, on each node kmaster, kworker1, kworker2

```
yum update
hostnamectl set-hostname kmaster # or, kworker1, kworker2     
hostnamectl --pretty set-hostname "Kubernetes master" # or, worker1, worker2
vi /etc/hosts  # Add "kmaster/kworker1/kworker2" at the end of each line
192.168.100.172 kmaster
192.168.100.173 kworker1
192.168.100.174 kworker2
```
Add user. I usually add this user as part of the Centos7 installation.
```
adduser jkozik
passwd jkozik
cd /root/.ssh
mkdir -p /home/jkozik/.ssh && chmod 700 /home/jkozik/.ssh
touch authorized_keys && chmod 600 authorized_keys
gpasswd -a jkozik wheel
```
Now verify that one can login using ssh to user jkozik.  Verify sudo works. 

Continuing as root, disable firewall, SElinux, swap. Set bridging for kubernetes
```
#firewall
systemctl disable firewalld; systemctl stop firewalld

#Swap
swapoff -a; sed -i '/swap/d' /etc/fstab

#SELinux
setenforce 0
sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux

#Bridging
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```
## Install Docker
Install git, docker, docker-compose.  Verify status and hello-world
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce git
git --version
systemctl start docker 
systemctl enable docker
systemctl status docker
```
Verify that docker works and then install docker compose
```
docker run hello-world
docker ps
curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
groupadd docker
usermod -aG docker jkozik
chmod 666 /var/run/docker.sock
```
## Verify docker
Login as jkozik. Verify that docker, docker-compose and git all work

```
git --version
docker run hello-world
git clone https://github.com/jkozik/SetupKubeadmCentos7
cd SetupKubeadmCentos7
docker-compose up
```
## Install kubernetes 
Install kubectl, kubeadm, kubelet on each of the nodes.
As root, do the following:
```
# Setup repository
cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubelet kubeadm kubectl
# Enable and start kubelet
systemctl enable --now kubelet
```
### For kmaster only
We've installed the same kubernetes software on three VMs -- now three nodes.  We now need to create a cluster.  Using the root prompt on the kmaster node, we need to initialize the cluster and create a network.
### Initialize the cluster
```
kubeadm init --apiserver-advertise-address=192.168.100.172 --pod-network-cidr=10.68.0.0/16
```
The output of this command will look like this.  This needs to be kept in order to join the other nodes in the cluster.
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.172:6443 --token 144c60.uv1po0x8xyflt055 \
        --discovery-token-ca-cert-hash sha256:72d91cedae66331277fada882c172b027974085388c1c084682bcc7821fcf60d
```
It is useful to run the following commands to give the status of the cluster.  We will re-run these commands as the cluster gets built out.
```
kubectl get nodes
kubectl get pods --all-namespaces
```
### install networking
The kubernetes architecture permits many different networking tools.  I am using Calico.  The steps to install Calico involve getting a yml file, editing for our ip address range and applying it.
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
vi calico.yaml  #uncomment the lines and set the CIDR to the one we used in the kubeadm init command
            #- name: CALICO_IPV4POOL_CIDR
            #  value: "10.68.0.0/16"
 export KUBECONFIG=/etc/kubernetes/admin.conf
 kubectl apply -f calico.yaml
 ```
 ### Verify kmaster setup
 Again, run the two status commands.  For me, I get the following:
 ```
[root@kmaster ~]# kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
kmaster   Ready    control-plane,master   16h   v1.21.1
[root@kmaster ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-78d6f96c7b-kxhtv   1/1     Running   0          14h
kube-system   calico-node-gvdxk                          1/1     Running   0          14h
kube-system   coredns-558bd4d5db-dtdzd                   1/1     Running   0          16h
kube-system   coredns-558bd4d5db-rmfhz                   1/1     Running   0          16h
kube-system   etcd-kmaster                               1/1     Running   0          16h
kube-system   kube-apiserver-kmaster                     1/1     Running   0          16h
kube-system   kube-controller-manager-kmaster            1/1     Running   0          16h
kube-system   kube-proxy-lg5qd                           1/1     Running   0          16h
kube-system   kube-scheduler-kmaster                     1/1     Running   0          16h
```
Following the output of the kubeadm command, setup kubernetes in a non-root login.
Logged in as jkozik
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Also, if needed, a new join command can be generated with the following:
```
kubeadm token create --print-join-command
```
# References

https://github.com/justmeandopensource/kubernetes/blob/master/docs/install-cluster-centos-7.md
https://stackoverflow.com/questions/57648829/how-to-fix-timeout-at-waiting-for-the-kubelet-to-boot-up-the-control-plane-as-st
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

# Youtube

Tech Tutorials
https://www.youtube.com/watch?v=gmUWZippIJE
Kubernetes (1.12.2) Installation On Centos7
https://www.youtube.com/watch?v=MgXh2HpNBtk
Install Kubernetes | Setup Kubernetes Step by Step | Kubernetes Training | Intellipaat
https://www.youtube.com/watch?v=l7gC4SgW7DU
Install kubeadm
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
Overview: Installing Kubernetes using Kubeadm | Coupon: UDEMYNOV20 | Udemy: Kubernetes Made Easy
https://www.youtube.com/watch?v=fT89wBxT1pI
Demo: Installing Kubernetes Using Kubeadm | Coupon: UDEMYNOV20 | Udemy: Kubernetes Made Easy
https://www.youtube.com/watch?v=lOd-0_DpYG8
Pods in Kubernetes | Coupon: UDEMYNOV20 | Udemy: Kubernetes Made Easy | Kubernetes Tutorial
https://www.youtube.com/watch?v=MgXh2HpNBtk
 Kube 1 ] Setup Kubernetes Cluster using Kubeadm on CentOS 7
https://www.youtube.com/watch?v=Araf8JYQn3w


