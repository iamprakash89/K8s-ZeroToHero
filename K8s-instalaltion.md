Agenda: Kubernetes Setup Using Kubeadm in AWS EC2 Ubuntu Servers
=======
Prerequisite:
=============
3 - Ubuntu Serves20.04
1 - Manager (4GB RAM, 2 Core) t2.medium
2 - Workers (1 GB, 1 Core) t2.small

Note: Open Required Ports in AWS Security Groups. For now, we will open All traffic.
==========COMMON FOR MASTER & SLAVES START ============
# First, login as ‘root’ user because the following set of commands need to be executed with ‘sudo’ permissions.
sudo su -

# Install Required packages and apt keys.
apt-get update -y; apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update -y

# Turn Off Swap Space
swapoff -a; sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

#############Install and configure container runtime####################
# Configure persistent loading of modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

# Load at runtime
sudo modprobe overlay; sudo modprobe br_netfilter

# Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload configs
sudo sysctl --system

# Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

# Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Install containerd
sudo apt update; sudo apt install -y containerd.io

# Configure containerd and start service
sudo mkdir -p /etc/containerd; sudo containerd config default|sudo tee /etc/containerd/config.toml

# restart containerd
sudo systemctl restart containerd; systemctl enable containerd; systemctl status  containerd

#Install kubeadm, Kubelet And Kubectl 
apt-get install -y kubelet kubeadm kubectl kubernetes-cni

# Enable and start kubelet service
systemctl daemon-reload; systemctl start kubelet; systemctl enable kubelet.service

==========COMMON FOR MASTER & SLAVES END==========

===========In Master Node Start====================
# Steps Only For Kubernetes Master
# Switch to the root user.
sudo su -

# Initialize Kubernates master by executing below commond.
kubeadm init

#exit root user & exeucte as normal user
exit

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# To verify, if kubectl is working or not, run the following command.
kubectl get pods -o wide --all-namespaces

#You will notice from the previous command, that all the pods are running except one: ‘kube-dns’. For resolving this we will install a # pod network. To install the weave pod network, run the following command:

# Download weavnet network plugin
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s-1.11.yaml
# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# Calio network plugin
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml

kubectl get nodes
kubectl get pods --all-namespaces


# Get token
kubeadm token create --print-join-command

#############Configure Aliases in .bashrc############### 
alias ku=kubectl
alias pods='kubectl get pods -A'
alias nodes='kubectl get nodes -o wide'

echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

#############Bash auto completion############### 
echo 'source <(kubectl completion bash)' >>~/.bashrc
=========In Master Node End====================

Add Worker Machines to Kubernates Master
=========================================
# Copy kubeadm join token from and execute in Worker Nodes to join to cluster
kubectl commands has to be executed in master machine.

Check Nodes 
=============
kubectl get nodes

Deploy Sample Application
==========================
kubectl run nginx-demo --image=nginx --port=80 
kubectl expose deployment nginx-demo --port=80 --type=NodePort

Get Node Port details 
=====================
kubectl get services

Add Labels to worker machine:
=====================
root@ip-10-2-1-112:~# kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
ip-10-2-1-112   Ready    control-plane   9m59s   v1.28.1
ip-10-2-1-35    Ready    <none>          7m16s   v1.28.1
ip-10-2-1-42    Ready    <none>          7m13s   v1.28.1
root@ip-10-2-1-112:~#
root@ip-10-2-1-112:~# kubectl label nodes ip-10-2-1-35 node-role.kubernetes.io/worker=true
node/ip-10-2-1-35 labeled
root@ip-10-2-1-112:~#
root@ip-10-2-1-112:~# kubectl label nodes ip-10-2-1-42 node-role.kubernetes.io/worker=truekubec	
node/ip-10-2-1-42 labeled
root@ip-10-2-1-112:~#
root@ip-10-2-1-112:~# kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
ip-10-2-1-112   Ready    control-plane   11m     v1.28.1
ip-10-2-1-35    Ready    worker          8m26s   v1.28.1
ip-10-2-1-42    Ready    worker          8m23s   v1.28.1
root@ip-10-2-1-112:~#
================
Set up Dashboard
================
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml


# Service Account & Cluster Role Binding
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: k8s-admin
  namespace: kube-system
  
 
Enable dashboard access from outside network
===============================================
nohup kubectl proxy --address 0.0.0.0 --accept-hosts ".*" & 
 
kubectl -n kube-system get service kubernetes-dashboard 
 
Edit Service Update type from clusterIp to NodePort
==================================================== 
kubectl -n kube-system edit service kubernetes-dashboard 
 
Get Bearer token
===================  
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep k8s-admin | awk '{print $1}')  

# Label node 
kubectl label node <nodeName>   node-role.kubernetes.io/worker=worker
