# TPhuc

ref: https://computingforgeeks.com/install-kubernetes-cluster-on-centos-with-kubeadm/

## Prepare - update os

master - master.tpk8s.com - 4r/2core \
node   - node01.tpk8s.com - 4r/2core

|Node   |Domain             |                 |
|-------|-------------------|-----------------|
| master|master.tpk8s.com   | ram 4 - 2 core  |
| node  | node01.tpk8s.com  | ram 2 - 2 core  |

```bash
sudo yum -y update && sudo systemctl reboots
```

## Install - Kubectl, Kubeadm, Kubelet

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

sudo yum clean all && sudo yum -y makecache
sudo yum -y install epel-release vim git curl wget kubelet kubeadm kubectl --disableexcludes=kubernetes
```
```bash
# check
[root@instance-1 ~]# kubeadm  version
kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.5", GitCommit:"804d6167111f6858541cef440ccc53887fbbc96a", GitTreeState:"clean", BuildDate:"2022-12-08T10:13:29Z", GoVersion:"go1.19.4", Compiler:"gc", Platform:"linux/amd64"}
[root@instance-1 ~]# kubectl version --client
WARNING: This version information is deprecated and will be replaced with the output from kubectl version --short.  Use --output=yaml|json to get the full version.
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.5", GitCommit:"804d6167111f6858541cef440ccc53887fbbc96a", GitTreeState:"clean", BuildDate:"2022-12-08T10:15:02Z", GoVersion:"go1.19.4", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
```

## Disable SELinux and Swap

https://www.cyberciti.biz/faq/disable-selinux-on-centos-7-rhel-7-fedora-linux/

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config

#Turn off swap.
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a

#Configure sysctl.
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Install docker runtime

```bash
# Install packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io -y

# Create required directories
sudo mkdir /etc/docker
sudo mkdir -p /etc/systemd/system/docker.service.d

# Create daemon json config file
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# Start and enable Services
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```

## Configure Firewalld

```bash
sudo systemctl disable --now firewalld
```

## Initialize your control-plane node

```bash
lsmod | grep br_netfilter

br_netfilter           22256  0 
bridge                151336  2 br_netfilter,ebtable_broute

# Enable kubelet service.
sudo systemctl enable kubelet

cat > /etc/containerd/config.toml <<EOF
[plugins."io.containerd.grpc.v1.cri"]
  systemd_cgroup = true
EOF

systemctl restart containerd

#Pull container images:
sudo kubeadm config images pull
```

## init

```bash
sudo kubeadm init \
>   --pod-network-cidr=192.168.0.0/16 \
>   --upload-certs \
>   --control-plane-endpoint=master.tpk8s.com
```

## Install network plugin

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml 
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```
