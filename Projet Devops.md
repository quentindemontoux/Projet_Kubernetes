# Projet Devops

Our goal is to setup following infrastructure:

| Hostname   | IP          |
| ---------- | ----------- |
| k8s-master | 192.168.1.7 |
| k8s-node   | 192.168.1.8 |

## Installation du cluster kubeadm on both vm

You'll need two vm. The first one will be the kube master and the second one the kube node.

We choose to create 2 vm on centos 7 distribution. 

Each one will need this hardware requirement: 2 CPU and 2 GB RAM.

Each one have one networking adapter branch on bridge and another one on host only.

### Step 1 : Set and update the hostname 

Add in /etc/hosts file for local name resolution. 

```bash
[root@localhost ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.56.7 k8s-master
192.168.56.8 k8s-node
[root@localhost ~]# sudo hostname k8s-<choose master or node>
```

### Step 2 : Disable swap 

```bash
[root@localhost ~]# cat /etc/fstab
#
# /etc/fstab
# Created by anaconda on Wed Dec  8 15:09:24 2021
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=d4c44571-008e-465e-84af-07dae47dda7d /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap 

[root@localhost ~]# swapoff -a
```

### Step 3 : Add firewall rules for Kubernetes service endpoint & kubelet on both vm

```bash
firewall-cmd --permanent --add-port=6443/tcp 
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --reload

[root@localhost ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: enp0s3 enp0s8
  sources: 
  services: dhcpv6-client ssh
  ports: 6443/tcp 10250/tcp 22/tcp
  protocols: 
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 

```

### Step 4 : Let iptables see bridged traffic

```bash
 cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf 
 net.bridge.bridge-nf-call-ip6tables = 1 
 net.bridge.bridge-nf-call-iptables = 1 
 EOF
 sysctl --system
```

### Step 5 : Add kubernetes repository

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo 
[kubernetes] 
name=Kubernetes 
baseurl=[https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch](https://packages.cloud.google.com/yum/repos/kubernetes-el7-\%24basearch) 
enabled=1 
gpgcheck=1 
repo_gpgcheck=1 
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg 
exclude=kubelet kubeadm kubectl 
EOF
```

### Step 6 : Set SElinux to enforcing mode 

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### Step 7 : Install yum utility 

```bash
 yum utility. yum install -y yum-utils device-mapper-persistent-data lvm2
```

### Step 8 : Add docker repository and install it 

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
yum update -y && yum install containerd.io docker-ce docker-ce-cli
```

### Step 9 : Create a overlay docker configuration file

```bash
 mkdir /etc/docker 
 cat > /etc/docker/daemon.json 
 <<
 EOF 
 { 
 "exec-opts": ["native.cgroupdriver=systemd"], 
 "log-driver": 
 "json-file", 
 "log-opts": { "max-size": "100m" }, 
 "storage-driver": 
 "overlay2", 
 "storage-opts": [ 
 "overlay2.override_kernel_check=true" 
 ] 
 } 
 EOF
```

### Step 10 : Add docker to systemd

```
mkdir -p /etc/systemd/system/docker.service.d 
systemctl daemon-reload 
systemctl restart docker 
systemctl enable docker
```

### Step 11 : Install kubeadm and other important packages

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes 
systemctl enable --now kubelet
```

## How to Deploy a Kubernetes Cluster

### Initialise the cluster only on kube-master

```bash
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.99.112 
```

Note : Here 192.168.99.112 is the ip of our hostonly adapter (enp0s8 for us). The **10.244.0.0/16** network value reflects the configuration of the *kube-flannel.yml* file. 

```bash
[root@k8s-master ~]# kubeadm init --apiserver-advertise-address 192.168.56.3 --pod-network-cidr=10.244.0.0/16

[...]
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

kubeadm join 192.168.56.7:6443 --token t0s0gu.oqx31nsmpz10sabl \
	--discovery-token-ca-cert-hash sha256:edd5b66f94f9473029de7ec294a1dba07533168b1296ddbc448a2b83c8efffc0 
```

```bash
 mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Pod Networking

#### Configuration de l'add-on en charge du réseau pour les pods au sein du cluster

Install Waves works for weaving containers into applications :

```bash
[root@kube-master ~]# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

#### Install auto-completion

```
yum install bash-completion 
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### Join the cluster only on kube-node

```bash
[root@kube-node ~]# kubeadm join 192.168.56.7:6443 --token t0s0gu.oqx31nsmpz10sabl \
> --discovery-token-ca-cert-hash sha256:edd5b66f94f9473029de7ec294a1dba07533168b1296ddbc448a2b83c8efffc0 
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

### Namespace

Like we're less than 10 persons to use our cluster, we decided to create only one namespace for the moment. Of course, we'll can create more later.

```bash

```





REF : https://www.youtube.com/watch?v=E3h8_MJmkVU

https://www.youtube.com/watch?v=E3h8_MJmkVU



## 

Création des Namespace et des ResourceQuotas pertinents pour votre cluster.

Il faut une organisation logique et expliquée.

Il est bien sûr possible de modifier l'organisation n'importe quand.

Il est même conseillé d'appliquer les quotas sur certains services après une utilisation sans quota afin d'avoir une idée de leur consommatio