# Vagrant Kubernetes bootstrap
This repo bootstraps 2 ubuntu VMs which is used to create Kubernetes.

# Pre-Requirements
manage VMs locally with Virtualbox and HashiCorp Vagrant
1. install virtualbox
```
brew install --cask virtualbox
```
for big sur, better download newest version from site https://www.virtualbox.org/wiki/Downloads

2. install vagrant
```
brew install --cask vagrant
```
or download from https://www.vagrantup.com/downloads
 
# Usage
## template
clone the repo and cd to dir where vagrantfile is

## create VMs
```
cd /Users/yourname/Documents/LocalHub/cka/local-kubernetes
vagrant up
```
This step creates 2 vms, configure them to run kubernetes.

## init master
#### Login to master
```
vagrant ssh master
sudo su -
```

#### Config proxy (China Only)
config proxy outside to allow/accelerate download.
on host machine, get local ip if using proxy on host, otherwise skip.
```
ipconfig getifaddr en1
```
on master&worker node
```
export https_proxy=http://ip:port
export http_proxy=http://ip:port
```

#### Installation preparation
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

apt-get update
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

#### Install Docker
Docker is most widely used container runtime. You can also find other container runtime in https://kubernetes.io/docs/setup/production-environment/container-runtimes/
##### installation docs
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker

https://docs.docker.com/engine/install/ubuntu/
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 apt-get update
 apt-get install docker-ce docker-ce-cli containerd.io
# apt-cache madison docker-ce
# apt-get install docker-ce=5:20.10.8~3-0~ubuntu-bionic docker-ce-cli=5:20.10.8~3-0~ubuntu-bionic containerd.io
```
Test docker
```
docker run hello-world
```
config storage
```
 cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
enable docker
```
 systemctl enable docker
 systemctl daemon-reload
 systemctl restart docker
```

#### Install kubeadm, kubelet and kubectl
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```
if downgrade needed, to latest v1.21
```
apt-cache madison kubeadm
apt-get update && apt-get install -y --allow-change-held-packages kubeadm=1.21.4-00 --allow-downgrades
apt-get update && apt-get install -y --allow-change-held-packages kubelet=1.21.4-00 kubectl=1.21.4-00 --allow-downgrades
```

#### Diable swap
```
echo net.bridge.bridge-nf-call-iptables = 1 >> /etc/sysctl.d/99-sysctl.conf
echo 1 >/proc/sys/net/bridge/bridge-nf-call-iptables
# disable swap in /etc/fstab and ran swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
swapoff -a
```

#### Unset proxy (China only)
```
unset http_proxy
unset https_proxy
```

#### Config kubelet
```
# vagrant only
# use eth1 instead of eth0
# for kubectl logs & exec cannot find
# https://medium.com/@joatmon08/playing-with-kubeadm-in-vagrant-machines-part-2-bac431095706
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
## add 
Environment="KUBELET_EXTRA_ARGS=--node-ip=master/worker-ip"
## before "ExecStart="
systemctl enable kubelet
systemctl daemon-reload
systemctl restart kubelet
```

#### Initiate Kubernetes Cluster
```
kubeadm init --apiserver-advertise-address=master-ip --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.21.4 --image-repository registry.aliyuncs.com/google_containers | tee /tmp/kubadmin.output
##Note: If 192.168.0.0/16 is already in use within your network you must select a different pod network CIDR, replacing 192.168.0.0/16 in the above command.
##Note2: can pull and tag related image accordingly beforehand, instead of using aliyun. latest images will be missed in aliyun
```
kubeconfig
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

#### Install network addon
Weavenet is my favourite, you can choose any.
find more in https://kubernetes.io/docs/concepts/cluster-administration/networking/#the-kubernetes-network-model
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

## init worker node
```
vagrant ssh node
sudo su - 
```
REPEAT_MASTER_NODE_STEPS_BEFORE_Initiate_Kubernetes_Cluster
THEN_RUN_THE_JOIN_COMMAND
```
## for fresh ones, from master && execute on worker node
cat /tmp/kubadmin.output
## or generate a new one
kubeadm token create --print-join-command
```
