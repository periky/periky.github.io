---
title: kubernetes集群部署(二):etcd集群
date: 2020-02-08 23:08:52
tags: kubernetes
---

## 安装etcd服务

```
wget https://github.com/etcd-io/etcd/releases/download/ \
    v3.3.17/etcd-v3.3.18-linux-arm64.tar.gz
tar xf etcd-v3.3.17-linux-amd64.tar.gz
cd etcd-v3.3.17-linux-amd64
mv etcd* /usr/bin/
```

<!--more-->

<br/>
<br/>

## 单节点部署

###### 配置文件

```
# etho是网卡名称
node_ip=$(ifconfig etho | grep inet | grep -v inet6 | awk '{print $2}')
cat << EOF | tee /etc/etcd/etcd.conf
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://${node_ip}:2380"
ETCD_LISTEN_CLIENT_URLS="http://${node_ip}:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://${node_ip}:2379"
EOF
```


###### systemctl service文件

```
cat << \EOF | tee /lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS}
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

<br/>
<br/>

## etcd集群

###### 配置文件

```
# master结点的IP地址
master_ip="10.0.2.15"
# etho是网卡名称, node_ip是etcd结点的IP地址
node_ip=$(ifconfig etho | grep inet | grep -v inet6 | awk '{print $2}')

cat << EOF | tee /etc/etcd/etcd.conf
ETCD_NAME="etcd01"　# {etcd01, etcd02, etcd03}, 结点名称
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://${node_ip}:2380"
ETCD_LISTEN_CLIENT_URLS="http://${node_ip}:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://${node_ip}:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${node_ip}:2380"
# node{num}_ip是每个结点的IP地址
ETCD_INITIAL_CLUSTER="etcd01=https://${master_ip}:2380,\
    etcd02=https://${node1_ip}:2380,etcd03=https://${node2_ip}:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

###### systemctl service文件

```
cat << \EOF | tee /lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
--name=${ETCD_NAME} \
--data-dir=${ETCD_DATA_DIR} \
--listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
--listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
--advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
--initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
--initial-cluster=${ETCD_INITIAL_CLUSTER} \
--initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
--initial-cluster-state=new
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
```

<br/>
<br/>

## etcd之HTTPS

*购买证书或者自制私有证书,使etcd支持https方式访问*

##### 安装配置CFSSL

*创建证书工具,用于提供https服务, 只需要在master结点上安装*

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

##### 创建证书

**生成根证书**

*只需在master结点上生成ca证书,然后同步至node结点中*

```
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```

```
# etcd ca配置文件
cat << EOF | tee ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shanghai",
            "ST": "Shanghai"
        }
    ]
}
EOF
```

```
# etcd server证书
# hosts是etcd结点ip列表
cat << EOF | tee server-csr.json
{
    "CN": "etcd",
    "hosts": [
        "10.0.2.15",
        "10.0.2.4",
        "10.0.2.5"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shanghai",
            "ST": "Shanghai"
        }
    ]
}
EOF
```

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=ca.pem \
    -ca-key=ca-key.pem -config=ca-config.json \
    -profile=www server-csr.json | cfssljson -bare server
```


##### 配置证书

*更新完master结点的service文件之后，同步到node结点*

编辑文件,`vim /lib/systemd/system/etcd.service`,在ExecStart指令后面添加如下启动参数

```
--cert-file=/etc/etcd/ssl/server.pem \
--key-file=/etc/etcd/ssl/server-key.pem \
--peer-cert-file=/etc/etcd/ssl/server.pem \
--peer-key-file=/etc/etcd/ssl/server-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem
```

编辑文件,`vim /etc/etcd/etcd.conf`, 更新协议, http -> https


## 服务健康检测

*默认检测的endpoint是http://0.0.0.0:2379*

```
etcdctl cluster-health
```
