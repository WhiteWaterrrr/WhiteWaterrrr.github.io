---
title: Kubesphere搭建-本地虚拟机
date: 2023-10-07 14:54:57
tags: 
    - kubesphere
    - K8S
cover: ..//img//kubesphere-build1//top.png
categories:
    - 跨域资源管理
---
本文描述了如何使用本地虚拟机搭建K8S以及kubesphere。
## 实验环境
|名称|value|
|---|---|
|操作系统|CentOS7.6|
|K8S版本|1.20.9|
|kubesphere版本|3.4|

|主机名|IP|CPU/内存|
|---|---|---|
|master|192.168.22.155|2/2g|
|node1|192.168.22.156|2/2g|
|node2|192.168.22.157|2/2g|

## K8S搭建
### 系统设置
所有机器都要操作
```BASH
# 关闭防火墙
systemctl stop firewalld

# 开机禁用防火墙自启：
systemctl disable firewalld.service

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 关闭swap
swapoff -a

# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# 修改hosts文件
echo "192.168.253.148  master" >> /etc/hosts
echo "192.168.253.149  node1" >> /etc/hosts
echo "192.168.253.150  node2" >> /etc/hosts
cat /etc/hosts

# 执行以下命令让设置生效
sysctl --system 

```
### Docker安装与启动
所有机器都要操作
```BASH
sudo yum remove docker*
sudo yum install -y yum-utils

#配置docker的yum地址
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo


#安装指定版本
sudo yum install -y docker-ce-20.10.7 docker-ce-cli-20.10.7 containerd.io-1.4.6

#	启动&开机启动docker
systemctl enable docker --now

# docker加速配置
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

```
### K8S安装与启动
所有机器都要操作
```BASH
#配置k8s的yum源地址
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF


#安装 kubelet，kubeadm，kubectl
sudo yum install -y kubelet-1.20.9 kubeadm-1.20.9 kubectl-1.20.9

#启动kubelet
sudo systemctl enable --now kubelet

```
master节点操作
```BASH
# 初始化K8S
# 该命令运行结束之后，会显示token，需要记录下来，用于后续node join操作
kubeadm init \
--apiserver-advertise-address=192.168.xxx.xxx \
--control-plane-endpoint=192.168.xxx.xxx \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.0.0.0/16
```

calico和k8s网络版本对应可以查看此[链接](https://blog.csdn.net/qq_32596527/article/details/127692734)，`k8s1.20.9`可以使用`calico3.20`
```BASH
# 安装网络插件
curl https://projectcalico.docs.tigera.io/archive/v3.20/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

node节点操作
```BASH
# 加入集群
kubeadm join k8s-master:6443 --token azfahe.n54ulrqmgtvyc71s \
    --discovery-token-ca-cert-hash sha256:5670d71391fa03b25a2d83ef2b13151ab07cf6ebd96d19b535536d92686d24bd
```
### K8S安装可能的问题
1. 在`kubeadm init`时，显示`/proc/sys/net/ipv4/ip_forward`不为1，[解决方法](https://www.cnblogs.com/wangbaobao/p/6674464.html)
****
## kubesphere搭建
### 配置默认存储类（nfs）
安装nfs
```BASH
# 所有机器安装nfs
yum install -y nfs-utils

# master：创建存放数据的目录，可以随意指定
mkdir -p /nfs

# master：配置挂载路径
echo "/nfs *(insecure,rw,sync,no_root_squash)" > /etc/exports

# master：开启nfs
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server

# master：使配置生效
exportfs -r

# master：检查配置是否生效
exportfs

# node：挂载，修改为master的ip
showmount -e 192.168.xxx.xxx
mkdir -p /nfs
mount -t nfs 192.168.xxx.xxx:/nfs /nfs
systemctl start nfs
```
以下步骤[参考](https://blog.csdn.net/qq_40722827/article/details/127948651)<br>
master：配置StorageClass（nfs-client.yaml），需要替换<br>
创建文件，把下方yaml内容写入文件
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 172.16.11.17   #替换成自己的nfs服务器
            - name: NFS_PATH
              value: /data/k8s  # 替换成自己的挂载目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.11.17   #替换成自己的nfs服务器
            path: /data/k8s  # 替换成自己的挂载目录

```
master：创建ServiceAccount（nfs-client-sa.yaml）
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```
master：创建StorageClass（nfs-client-class.yaml）
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs
```
master：创建资源对象
```BASH
kubectl create -f nfs-client.yaml
kubectl create -f nfs-client-sa.yaml
kubectl create -f nfs-client-class.yaml

# 设置course-nfs-storage 的 StorageClass 为 Kubernetes 的默认存储后端
kubectl patch storageclass course-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 查看storageclass，有default字样
kubectl get storageclass
```
![查看storageclass](.\kubesphere-build1\1.png)

### 配置metrics-server
参考[链接](https://www.cnblogs.com/vipsoft/p/16896510.html)

### 安装kubesphere
官网[链接](https://kubesphere.io/zh/docs/v3.4/quick-start/minimal-kubesphere-on-k8s/)<br>
```bash
vi storageclass.yaml
#添加以下内容
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer 

vi persistentVolumeClaim.yaml
#添加以下内容
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pve
spec:
  accessModes:
     - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: local-storage
```
```BASH
# 下载核心文件
wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.0/kubesphere-installer.yaml
wget https://github.com/kubesphere/ks-installer/releases/download/v3.4.0/cluster-configuration.yaml

# 安装
kubectl apply -f storageclass.yaml
kubectl apply -f persistentVolumeClaim.yaml

kubectl apply -f kubesphere-installer.yaml
kubectl apply -f cluster-configuration.yaml

# 使用命令查看进度
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f 
# 检查控制台的端口（默认为 30880）：
kubectl get svc/ks-console -n kubesphere-system
```
### 通过Web进行访问
请确保服务器的30880可以访问<br>
默认帐户和密码`admin` `P@88w0rd`
![实验结果](.\kubesphere-build1\2.png)