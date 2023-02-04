---
layout: post
title: "使用kubeadm安装生产级k8s集群"
subtitle: ""
date: 2023-02-04
author: "Chaos"
header-img: "img/genshin-bg.jpeg"
tags: 
  - kubernetes
  - k8s
  - kubeadm
  - containerd
  - linux
---


## 预先步骤

｜ 目标安装版本 kubernetes 1.24.3
### 服务器配置：


|        | CPU   | Memory | Disk | System | Ip |
| :----: | :---: | :---:| | :---:| :---:| :---: |
| k8s-node-1 | 2C | 8G | 100GB | Centos 7.9| 192.168.50.50|
| k8s-node-1 | 2C | 8G | 100GB | Centos 7.9| 192.168.50.51|
| k8s-node-1 | 2C | 8G | 100GB | Centos 7.9| 192.168.50.52|

## 1 配置Linux，安装依赖 

### 1.1 update yum package到最新
`yum update -y`
### 1.2 关闭Swap，k8s worker node 禁止使用swap

shell 执行
```shell
swapoff  -a
```
编辑 /etc/fstab, 注释swap配置
```
# /dev/mapper/centos-swap swap swap    defaults        0 0
```

### 1.3 关闭firewalld
```shell
systemctl disable firewalld
systemctl stop firewalld
```

### 1.4 设置SELINUX
```shell
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 1.5 设置node hostname
```shell
hostnamectl set-hostname k8s-node-1
```
### 1.6 开启ip_forward
```shell
echo 1 > /proc/sys/net/ipv4/ip_forward
```
### 1.7 开启bridge-nf-call-iptables
```shell
modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### 1.8 /etc/hosts 配置，保证node 间可以通过hostname互相访问
`vim /etc/hosts`
```shell
k8s-node-1 192.168.50.50
k8s-node-2 192.168.50.51
k8s-node-3 192.168.50.52
```

### 1.9 重启机器
```shell
reboot
```
## 2 运行时（Containerd）安装
因为K8s在1.24以后已经不再支持docker作为容器运行时，所以我们选择安装containerd作为容器运行时

### 2.1 通过yum安装Containerd
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install containerd -y
systemctl enable containerd
systemctl restart containerd
```

### 2.2 修改containerd的配置
```shell
containerd config default > /etc/containerd/config.toml
```
修改/etc/containerd/config.toml,需要修改两个地方
#### 2.2.1 修改cgroup为systemd
SystemdCgroup = true
![systemd](/img/install-k8s-with-kubeadm/containerd-cgroup.png)
#### 2.2.2 修改sandbox_image
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
![sandbox_image](/img/install-k8s-with-kubeadm/containerd-sandbox-image.png)
保存 config.toml后退出

### 2.3 重启containerd
```
systemctl daemon-reload
systemctl restart containerd
```

## 3 安装k8s
### 3.1 配置kubernetes 源
```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.2 安装kubeadm , kubelet, kubectl
```shell
yum install -y kubelet-1.24.3 kubeadm-1.24.3 kubectl-1.24.3 --disableexcludes=kubernetes
```

### 3.3 配置kubeadm config yaml, 示例配置如下， 保存为kubeadmin-config.yaml
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.50.50 ## 替换为master ip 
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock ## containerd sock
  imagePullPolicy: IfNotPresent
  name: k8s-node-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: nuc-k8s
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.24.3
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16 # pod 子网，需要后续和cni 插件的subnet 保持一致
  serviceSubnet: 10.96.0.0/12 
scheduler: {}
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd # cgroup 驱动，与containerd cgroup 驱动保持一致
```

### 3.4 安装前预先拉取镜像
```shell
kubeadm config images pull --config kubeadm-config.yaml
```
拉取完成后就可以执行安装了

### 3.5 执行安装
```shell
kubeadm init --config kubeadm-config.yaml
```
安装完成后 可以看到从日志中看到如下类似的信息
```shell
kubeadm join 192.168.50.50:6443 --token xxxx --discovery-token-ca-cert-hash sha256:xxxxx
```

复制保存，在其他node 上执行上面的命令，就可以加入这个集群了
** 至此集群已经初步搭建完成 **
但是 如果你执行 `kubectl get nodes` 或者 `kubectl get po -A`, 你会发现 workernode 还处于 unready 状态， 很多pod 比如 kubedns 还处于pending 状态，那是因为我们还有最后一步要做，安装cni 插件, 在此之前，先配置kubeconfig

#### 3.5.1 kubeconfig 配置
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.6 安装cni插件，以calico为例
#### 3.6.1 下载calico yaml 配置
```shell
curl https://docs.projectcalico.org/manifests/calico.yaml
```
yaml中有一个地方需要我们修改

```yaml
            - name: CALICO_IPV4POOL_CIDR
              value: "10.244.0.0/16" ## 此处需与kubeadm podsubnet配置保存一致
```
![calico](/img/install-k8s-with-kubeadm/calico-config.png)

```shell
kubectl apply -f calico.yaml
```

## 4 安装完成