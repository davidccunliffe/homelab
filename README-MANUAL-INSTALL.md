# Installing Kubernets 1.28 on Ubuntu 24.04 with Calico and ContainerD

This step-by-step guide on this page will explain how to install a Kubernetes (v1.28) Cluster on Ubuntu 22.04 using Kubeadm commands step by step. I'll eventually post a guide on deploying this through ansible at a later date.

## Prerequisites

In this guide, we are using one master node and two worker nodes. Following are system requirements on each node, per Kubernetes website.

- Minimal install Ubuntu 22.04
- Minimum 2GB RAM or more
- Minimum 2 CPU cores / or 2 vCPU
- 20 GB free disk space on /var or more
- Sudo user with admin rights
- Internet connectivity on each node

## Lab Setup

- Master Node:  192.168.189.6  -  k8sMaster.lan
- First Worker Node:  192.168.189.7  -  k8sWorker1.lan
- Second Worker Node:  192.168.189.8  -  k8sWorker2.lan
- Third Worker Node: 192.168.189.9  -  k8sWorker3.lan

## Set hostname on each Node

### Login to master node and set hostname via hostnamectl command.
```bash
sudo hostnamectl set-hostname "k8sMaster.lan"
exec bash
```

### On the worker nodes, run:
```bash
sudo hostnamectl set-hostname "k8sWorker1.lan"
exec bash
sudo hostnamectl set-hostname "k8sWorker2.lan"
exec bash
sudo hostnamectl set-hostname "k8sWorker3.lan"
exec bash
```

### Add the following to each node:
```bash
sudo tee /etc/hosts <<EOF
192.168.189.6    k8sMaster.lan k8smaster
192.168.189.7    k8sWorker1.lan k8sworker1
192.168.189.8    k8sWorker2.lan k8sworker2
192.168.189.9    k8sWorker3.lan k8sworker3
EOF
```

## Disable Swap and Add Kernel Parameters

### Execute following commands to disable swap and prevent fstab from reloading
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Load the following kernel modules on all the nodes
```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Set the following Kernel parameters for Kubernetes:
```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT
```

### Reload the changes from above:
```bash
sudo sysctl --system
```

## Install Containerd Runtime

### Install the following packages to ensure all dependencies are completed
```bash
sudo apt update && sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

### Enable docker repository
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

### Install containerd
```bash
sudo apt update && sudo apt install -y containerd.io
```

### Configure containerd so that it starts using systemd as cgroup
```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

### Restart and enable containerd service
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install Kubernetes

### Add Repository Keys for Kubernetes
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Add Apt Repository for Kubernetes v1.28
```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes Tools
```bash
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize the cluster
> After you Initialize the cluster, you will see a message with instructions on how to join worker nodes to the cluster. Make a note of the kubeadm join command for future reference.
```bash
sudo kubeadm init --control-plane-endpoint=k8smaster.lan
```


### Setup kubectl for current user
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Test access to the cluster
```bash
kubectl cluster-info
kubectl get nodes
```

### Join the worker nodes to the cluster
> This is just an example command

```bash
kubeadm join k8sMaster.lan:6443 --token b1gn14.39lh83lkzvi9j7ww \
        --discovery-token-ca-cert-hash sha256:5be9dea5e560c4cbxxxxecd27cc6e1c31e90ae953cb48c1e7d702e3d2e2xxxx
```

### Check if workers are in the cluster `Done on Master` but will not be ready yet due to no CNI being deployed
```bash
kubectl get nodes 
```

## Installing Calico Network Plugin

### Applying the Pod Manifest
> A network plugin is required to enable communication between pods in the cluster. Run the following kubectl command to install Calico network plugin from the master node.
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

### Verify the status of the pods in kube-system namespace
```bash
kubectl get pods -n kube-system -o wide
```

### Check node status
```bash
kubectl get nodes -o wide
```

## Test your Kubernetes Cluster Install now

### Will use nginx to test the installation, by creating a deployment called nginx-app
```bash
kubectl create deployment nginx-app --image=nginx --replicas=2
```

### Check the status of nginx-app deployment
```bash
kubectl get deployment nginx-app
```

### Expose the deployment as Nodeport on port 80
```bash
kubectl expose deployment nginx-app --type=NodePort --port=80
```

### Check Service status
```bash
kubectl get svc nginx-app
kubectl describe svc nginx-app

NAME        TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx-app   NodePort   10.105.125.201   <none>        80:31844/TCP   3m23s
Name:                     nginx-app
Namespace:                default
Labels:                   app=nginx-app
Annotations:              <none>
Selector:                 app=nginx-app
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.125.201
IPs:                      10.105.125.201
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31844/TCP
Endpoints:                172.16.123.1:80,172.16.202.1:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### Curl a worker node on port detailed in the describe step `NodePort`
```bash
curl http://localhost:<NODEPORT> #on cluster
curl http://<node-server-ip>:<NODEPORT> #off cluster
```

### Cleanup deployment
```bash
 kubectl delete deployment nginx-app
```
