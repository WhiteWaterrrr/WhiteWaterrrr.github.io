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
- 使用内网ip+iptable对数据进行转发
- 创建虚拟网卡，配置flannel

## 1.内网ip+iptable
实验[参考](https://blog.csdn.net/qq_43285879/article/details/120794910#:~:text=%E4%B8%80%E4%B8%AAk8s%E9%9B%86%E7%BE%A4%E2%80%94%E2%80%94%E8%B7%A8%E4%BA%91%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%83%A8%E7%BD%B2%201%201%E3%80%81%E5%AE%89%E8%A3%85Docker%20sudo%20yum%20remove%20docker%2A%20sudo,Bash%E5%A4%8D%E5%88%B6%E4%BB%A3%E7%A0%81%20%23%20%E5%9C%A8%E6%AF%8F%E4%B8%AA%E6%9C%BA%E5%99%A8%E3%80%82%20yum%20install%20-y%20nfs-utils%20)
#### 实验结果
```bash
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS     ROLES                  AGE     VERSION
k8s-master   Ready      control-plane,master   7d13h   v1.20.9
k8s-node1    Ready      <none>                 16s     v1.20.9
```
上述实验结果看似正确，但是当我们使用`kubectl get pods -n kube-system`命令时，会发现CNI（calico/Flannel）有关连接其他云服务器的pods一直不在ready状态。

此方案的问题有在[此处](https://blog.csdn.net/weixin_43988498/article/details/122639595)提及，我还没有想到解决方案~.~

#### 实验准备
服务器：ucloud 2核2G+青云 2核2G

操作系统：CentOS7

#### 实验过程修正
安装Calico网络插件时需要注意calico版本，k8s1.20.9适配于calico 3.20
```bash
curl https://docs.projectcalico.org/archive/v3.20/manifests/calico.yaml -O
```

## 2.创建虚拟网卡，配置flannel
实验[参考](https://blog.csdn.net/weixin_43988498/article/details/122639595)

# 未完待续