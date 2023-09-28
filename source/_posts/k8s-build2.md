---
title: K8S集群搭建-跨云跨VPC
date: 2023-09-25 20:08:51
tags: 
    - K8S
cover: ..//img//k8s-build2//top.png
categories:
    - 跨域资源管理
---

本文描述了如何使用不同云厂商的服务器搭建K8S集群，供下次再次搭建集群时参考！

跨云跨VPC的K8S集群搭建和局域网K8S集群搭建有什么区别，参考这篇[文章](https://zhuanlan.zhihu.com/p/106793809)


## 跨云解决方案概览
搭建跨云K8S集群有以下方法：
- 创建虚拟网卡，配置flannel（2023-9-28实现此方法）
- 使用内网ip+iptable对数据进行转发（本文还未成功实现，附在文末）

***

## 创建虚拟网卡，配置flannel
实验[参考](https://blog.51cto.com/xiaowangzai/5167661)
### 实验准备
1. 服务器：两台内网不通的云服务器，本实验配置为2核2G内存
2. 操作系统：CentOS7.6
3. Dokcer版本：20.10.7（具体查看[参考](https://blog.51cto.com/xiaowangzai/5167661)）
4. K8S版本：1.20.9
### 教程过程修正

***
1. 在调整内核参数时，需要增加这一语句
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
#以下为新增 开启转发
net.ipv4.ip_forward = 1
EOF
```

2. 安装flannel网络插件时，执行以下语句，之后对`kube-flannel.yml`文件进行修改
```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

3. 实验验证
```bash
#在kubenetes集群中创建一个pod创建nginx
kubectl create deployment nginx --image=nginx
#暴露Nginx端口,创建一个svc
kubectl expose deployment nginx --port=80 --type=NodePort
#查看Nginx端口
kubectl get pod,svc
```
之后你便可以在浏览器上进行访问，`<公网ip>:3xxxx`，若出现以下界面，则跨云搭建K8S集群成功。
![实验结果](..\\img\\k8s-build2\\r.png)

## 内网ip+iptable
实验[参考](https://blog.csdn.net/qq_43285879/article/details/120794910#:~:text=%E4%B8%80%E4%B8%AAk8s%E9%9B%86%E7%BE%A4%E2%80%94%E2%80%94%E8%B7%A8%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%83%A8%E7%BD%B2%201%201%E3%80%81%E5%AE%89%E8%A3%85Docker%20sudo%20yum%20remove%20docker%2A%20sudo,Bash%E5%A4%8D%E5%88%B6%E4%BB%A3%E7%A0%81%20%23%20%E5%9C%A8%E6%AF%8F%E4%B8%AA%E6%9C%BA%E5%99%A8%E3%80%82%20yum%20install%20-y%20nfs-utils%20)
### 实验结果
```bash
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS     ROLES                  AGE     VERSION
k8s-master   Ready      control-plane,master   7d13h   v1.20.9
k8s-node1    Ready      <none>                 16s     v1.20.9
```
上述实验结果看似正确，但是当我们使用`kubectl get pods -n kube-system`命令时，会发现CNI（calico/Flannel）有关连接其他云服务器的pods一直不在ready状态。

此方案的问题有在[此处](https://blog.csdn.net/weixin_43988498/article/details/122639595)提及，我还没有想到解决方案~.~

### 实验准备
服务器：ucloud 2核2G+青云 2核2G

操作系统：CentOS7

### 实验过程修正
安装Calico网络插件时需要注意calico版本，k8s1.20.9适配于calico 3.20
```bash
curl https://docs.projectcalico.org/archive/v3.20/manifests/calico.yaml -O
```
