# homelab

This is my homelab. Ansible setup for my homelab. Still a work in progress.

## Setup

Download this repo to the control machine. Then run the following commands:

```bash
git clone https://github.com/2wdavidcunliffe/homelab.git
cd homelab
./ubuntu-ansible-bootstrap.sh
ansible-playbook ./ansible/k8s-master.yml --ask-pass -i ./ansible/inventory/lab/hosts

```

## Add the kube configs to the user account

```bash
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


## Reset cluster for testing

Login to root and run the following commands:

```bash
kubectl delete -f https://docs.projectcalico.org/manifests/calico.yaml
kubeadm reset -f
rm -rf /etc/cni/net.d
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
systemctl stop kubelet
systemctl stop containerd
iptables --flush
iptables -tnat --flush
systemctl start kubelet
systemctl start containerd
rm -rf $HOME/.kube

```

## TODO

- Need to rebuild the ISO for ubuntu so I don't have to keep typing autoinstall
  -
- Need to get a playbook which will setup the ansible user
  - create a ssh key that will be copied to all the systems
  - create a user ansible and add it to the sudoers file
  - generate a assign a ssh key to the ansible user
- Need a playbook which will reset the cluster and setup the cluster
  - it will login using the ansible user
  - it will reset the master and the nodes
  - it will setup the cluster
- Need a playbook which will setup the kubernetes dashboard
  - it will login using the ansible user
  - it will setup the dashboard
