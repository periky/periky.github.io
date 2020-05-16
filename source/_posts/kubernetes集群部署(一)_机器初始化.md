---
title: kubernetes集群部署(一):初始化
date: 2020-02-05 13:08:40
tags: kubernetes
---

## 初始化

**关闭防火墙和SELINUX**

```
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i s/SELINUX=enforcing/SELINUX=disabled/ /etc/selinux/config
```

<!--more-->

## kubernetes结点初始化

**关闭swap**

```
swapoff -a && sysctl -w vm.swappiness=0
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**安装docker**

```
curl https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
yum install docker-ce-18.06.2.ce-3.el7 -y
systemctl start docker && systemctl enable docker
```

**内存参数**

```
modprobe br_netfilter

cat << EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
EOF

sysctl -p /etc/sysctl.d/k8s.conf
```