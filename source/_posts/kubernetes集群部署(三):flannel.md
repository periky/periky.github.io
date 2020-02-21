---
title: kubernetes集群部署(三):flannel集群
date: 2020-02-09 13:08:40
tags: kubernetes
---

**安装flannel服务**

```
wget https://github.com/coreos/flannel/releases/download/v0.11.0/ \ 
    flannel-v0.11.0-linux-amd64.tar.gz
mv flanneld mk-docker-opts.sh /usr/bin/
```

<!--more-->

**配置文件**

```
mkdir /etc/flannel
cat <<EOF > /etc/flannel/flannel.conf
# --etcd-endpoints是etcd结点连接信息
FLANNEL_OPTIONS="--etcd-endpoints=https://10.0.2.15:2379, \
    https://10.0.2.4:2379,https://10.0.2.5:2379 \
    -etcd-cafile=/etc/etcd/ssl/ca.pem \
    -etcd-certfile=/etc/etcd/ssl/server.pem \
    -etcd-keyfile=/etc/etcd/ssl/server-key.pem"
EOF
```


**systemctl service文件**

```
cat << \EOF | tee /lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target etcd.service
Before=docker.service
[Service]
Type=notify
EnvironmentFile=/etc/flannel/flannel.conf
ExecStart=/usr/bin/flanneld --ip-masq $FLANNEL_OPTIONS
ExecStartPost=/usr/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

**修改docker.service文件**

```
添加: EnvironmentFile=/run/flannel/subnet.env
# $ARGS是docker启动参数
ExecStart命令更新: /usr/bin/dockerd $DOCKER_NETWORK_OPTIONS $ARGS
```

**向etcd中写入POD网段信息**

*flannel读取etcd中的pod网段信息,用于设置docker0网桥*

```
etcdctl \
--ca-file=/etc/etcd/ssl/ca.pem --cert-file=/etc/etcd/ssl/server.pem \
--key-file=/etc/etcd/ssl/server-key.pem \
--endpoints="https://10.0.2.15:2379" \
set /coreos.com/network/config  '{ "Network": "172.20.0.0/16", "Backend": {"Type": "vxlan"}}'
```