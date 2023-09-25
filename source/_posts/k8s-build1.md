---
title: K8S集群搭建-局域网内
date: 2023-09-22 16:03:19
tags: K8S
---

本文描述了如何从本地虚拟机搭建K8S集群，供下次再次搭建集群时参考！搭建K8S集群的文章在中文网络很多，并且基本都是使用本地虚拟机进行搭建。

本文[参考](https://blog.csdn.net/qq_45617555/article/details/130395158)。

## 环境准备
- 虚拟机平台：VMware17
- 操作系统：CentOS7
- 虚拟机设置：三台除主机名和网络以外均一致的虚拟机（可以通过VMware的克隆操作得到）
- 网络配置：NAT，三台虚拟机可以PING通

## 注意
本次搭建基本参考此[链接](https://blog.csdn.net/qq_45617555/article/details/130395158)，在安装过程中会出现以下问题：

#### kubelet无法启动

三台虚拟机在执行完以下语句之后会出现错误。
- 错误原因：在启动kubelet服务之前，master节点首先需要`kubeadm init`，也就是先进行后续操作之后再进行此操作。
- 解决方法：先让master节点进行`kubeadm init`操作，再对三台服务器进行本操作，启动kubelet服务。

```bash
systemctl start kubelet #启动kubelet服务 
systemctl status kubelet #查看kubelet服务是否正常启动成功
```

#### K8S操作需要等待
以下语句执行后需要等待调度，K8S才会就绪
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml 

kubeadm join 192.168.XXX.XXX:6443 --token 6509w1.4nu34gycu80kl4h5 \
    --discovery-token-ca-cert-hash sha256:9560eb6f7201c8032469a4204f33b0e0dd83a7118ede09d3b04c6d4ab2d723d2

```

## 验证
验证时，如果没有图形化界面，可以通过本机浏览器去访问虚拟机ip。

## 重启
重启之后需要重新运行docker
```bash
systemctl start docker
```
重启之后可能会遇到问题，执行以下操作查看各个pod的运行情况
```bash
kubectl get pods --all-namespaces -o wide
```
如果要将node重新join入集群，需要现在master上面重新获取有效的令牌，使用以下语句生成join命令
```bash
kubeadm token create --print-join-command
```
