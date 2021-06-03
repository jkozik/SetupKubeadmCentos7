# Setup kubeadm on Centos7
To setup Kubernetes on my home server, I need to create 3 nodes.  A master and two workers.  There are several procedures for getting this working (referenced below).  It turns out I had to look at several of them to finally get things working, so therefore, I am writing this note to help me keep things straight.

I am using VirtualBox and creating 3 VMs all based on the Centos7 minumum iso.  I install them from scratch to a static IP addresses on my home LAN (192.168.100.0/24). These VMs are setup with a bridged interface and can ping anything in my home LAN or Internet.  

Kubernetes is setup in a cluster of VMs -- each VM is called a node.  kubeadm is the tool that links the nodes. kubectl is the tool to control the cluster.  The nodes run pods which are made of docker containers. The pods network across the cluster using a tool called Calico. The steps below setup the Centos7 environment; install the main docker and kubernetes tools.  The node called kmaster is the controller node.  The two other nodes, kworker1, kworker2 run the applications in pods.  

The setup for the kmaster and the kworker1/2 nodes are the same until the cluster is created.  The kubeadm tool initializes a cluster and creates a network.  The tool generates a join script that the kworker1/2 nodes run to join the cluster. The kubectl command is used to verify that the cluster setup was completed successfully.

Most of the steps below are common to all three nodes.

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
# adduser jkozik
# passwd jkozik
# cd /root/.ssh
# mkdir -p /home/jkozik/.ssh && chmod 700 /home/jkozik/.ssh
# cp authorized_keys /home/jkozik/.ssh && chmod 600 authorized_keys
# chown -R jkozik:jkozik /home/jkozik/.ssh
# gpasswd -a jkozik wheel
```
Now verify that one can login using ssh to user jkozik.  Verify sudo works. 

Continuing as root, disable firewall, SElinux, swap. Set bridging for kubernetes
```
#firewall
systemctl disable firewalld; systemctl stop firewalld
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

Install git, docker, docker-compose.  Verify status and hello-world
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce git
git --version
systemctl restart docker && systemctl enable docker
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
Login as jkozik. Verify that docker, docker-compose and git all work

```
git --version
docker run hello-world
git clone https://github.com/jkozik/SetupCentOS7
cd SetupKubeadmCentos7
docker-compose up
```

