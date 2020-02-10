---
title: kubernetes机器部署(四):k8s集群部署
date: 2020-02-09 15:08:40
tags: kubernetes
---

## 获取二进制文件

*所有操作都在master结点上执行*

**获取安装文件**

```
wget https://dl.k8s.io/v1.15.5/kubernetes-server-linux-amd64.tar.gz
tar xf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-controller-manager kubectl /usr/bin
cp kube-scheduler kubelet kube-proxy  /usr/bin
```


<!--more-->

## kube-apiserver服务

**配置文件**

```
# vim /etc/kubernetes/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=4 \
--etcd-servers=http://${master_ip}:2379 \
--bind-address=${master_ip} \
--secure-port=6443 \
--advertise-address=${master_ip} \
--allow-privileged=true \
--service-cluster-ip-range=10.40.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth \
--token-auth-file=/etc/kubernetes/token.csv \
--service-node-port-range=30000-50000"
```

**systemctl service文件**

```
cat << \EOF > /lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=flannld.service
[Service]
EnvironmentFile=/etc/kubernetes/kube-apiserver.conf
ExecStart=/usr/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```


**启动kube-apiserver服务**

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
```


## kube-controller-manager服务

**配置文件**

```
# vim /etc/kubernetes/kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=4 \
--master=127.0.0.1:8080 \
--leader-elect=true \
--address=127.0.0.1 \
--service-cluster-ip-range=10.40.0.0/24 \
--cluster-name=kubernetes"
```

**systemctl service文件**

```
cat << \EOF > /lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
[Service]
EnvironmentFile=-/etc/kubernetes/kube-controller-manager.conf
ExecStart=/usr/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

**启动kube-controller-manager服务**

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```


## kube-scheduler服务

**配置文件**

```
# vim /etc/kubernetes/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--logtostderr=false --log-dir=/var/log/kubernetes \
 --v=4 --master=127.0.0.1:8080 --leader-elect"
```

**systemctl service文件**

```
cat << \EOF > /lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=kube-apiserver.service
[Service]
EnvironmentFile=-/etc/kubernetes/kube-scheduler.conf
ExecStart=/usr/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

**启动kube-scheduler服务**

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```


## kubernetes Token

**TLS Bootstrapping Token**

```
# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
449bbeb0ea7e50f321087a123a509a19

# vim /cloud/k8s/kubernetes/cfg/token.csv
449bbeb0ea7e50f321087a123a509a19,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

**kube-bootstrap.kubeconfig**

```
BOOTSTRAP_TOKEN=449bbeb0ea7e50f321087a123a509a19
KUBE_APISERVER="https://${master_ip}:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

**kube-proxy.kubeconfig**

```
BOOTSTRAP_TOKEN=449bbeb0ea7e50f321087a123a509a19
KUBE_APISERVER="https://${master_ip}:6443"
# 创建kube-proxy kubeconfig文件
kubectl config set-cluster kubernetes \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```


**绑定kubelet-bootstrap用户到集群**

```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```


## kubelet服务

**配置文件**

```
# node-labels=node-role.kubernetes.io/master, master结点使用该参数
# node-labels=node-role.kubernetes.io/node, node结点使用该参数
# vim /etc/kubernetes/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=4 \
--node-labels=node-role.kubernetes.io/master \
--hostname-override=k8s-master \
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
--config=/etc/kubernetes/kubelet.config \
--cert-dir=/etc/kubernetes/ssl \
--cluster-dns=10.40.0.5 \
--cluster-domain=cluster.domain \
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/\
google-containers/pause-amd64:3.0"
```

**kubelet配置模板**

```
# vim /etc/kubernetes/kubelet.config
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: ${node_ip}
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
clusterDomain: cluster.local.
failSwapOn: false
authentication:
  anonymous:
    enabled: true
```

**systemctl service文件**

```
cat << \EOF > /lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service
[Service]
EnvironmentFile=/etc/kubernetes/kubelet.conf
ExecStart=/usr/bin/kubelet $KUBELET_OPTS
Restart=on-failure
KillMode=process
[Install]
WantedBy=multi-user.target
EOF
```

**启动kubelet服务**

```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```


## kube-proxy服务

**配置文件**

```
# vim /etc/kubernetes/kube-proxy.conf
KUBE_PROXY_OPTS="--logtostderr=true \
--v=4 \
--hostname-override=k8s-master \
--cluster-cidr=10.40.0.0/24 \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig"
```

**systemctl service文件**

```
cat << \EOF > /lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=-/etc/kubernetes/kube-proxy.conf
ExecStart=/usr/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

**启动kube-proxy服务**

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
```


## CoreDns

```
git clone https://github.com/coredns/deployment.git
cd kubernetes
yum -y install jq
./deploy.sh -r 10.40.0.0/24 -i 10.40.0.5 -d cluster.local |kubectl apply -f -
```


## kubernetes机器提供HTTPS服务

**安装配置CFSSL**

*创建证书工具,用于提供https服务*

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```


**cs根证书**

```
cat << EOF | tee /etc/kubernetes/ssl/ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
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
cat << EOF | tee /etc/kubernetes/ssl/ca-csr.json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shanghai",
            "ST": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
# 生成根证书
cd /etc/kubernetes/ca
cfssl gencert -initca /etc/kubernetes/ssl/ca-csr.json | cfssljson -bare ca -
```

**kube-apiserver证书**

```
# master结点主机名
MASTER_HOSTNAME=$(/bin/hostname)
CLUSTER_KUBERNETES_SVC_IP=$(kubectl get svc kubernetes  | \
    awk '{print $3}' | tail -n1)
cat << EOF | tee /etc/kubernetes/ssl/server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "$MASTER_HOSTNAME", # master结点主机名
        "$CLUSTER_KUBERNETES_SVC_IP", # 
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shanghai",
            "ST": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
# 生成apiserver的证书
cd /etc/kubernetes/ssl/
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem -ca-key=/etc/kubernetes/ssl/ \
    ca-key.pem -config=/etc/kubernetes/ssl/ca-config.json -profile=kubernetes \
    /etc/kubernetes/ssl/server-csr.json | cfssljson -bare server
```

**kube-apiserver的配置文件**

*添加以下配置信息到apiserver的配置文件中,使apiserver提供https服务*

```
--tls-cert-file=/etc/kubernetes/ssl/server.pem  \
--tls-private-key-file=/etc/kubernetes/ssl/server-key.pem \
--client-ca-file=/etc/kubernetes/ssl/ca.pem \
--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem
```

**kube-controller-manager的配置文件**

*添加以下配置信息到kube-controller-manager的配置文件中*

```
--cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/etc/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem"
```

**kube-proxy证书**

```
cat << EOF | tee /etc/kubernetes/ssl/kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shanghai",
      "ST": "Shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

```
# 生成kube-proxy证书
cd /etc/kubernetes/ssl/
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
-ca-key=/etc/kubernetes/ssl/ca-key.pem \
-config=/etc/kubernetes/ssl/ca-config.json \
-profile=kubernetes /etc/kubernetes/ssl/kube-proxy-csr.json \
| cfssljson -bare kube-proxy
```

**更新kubeconfig脚本**

```
BOOTSTRAP_TOKEN=449bbeb0ea7e50f321087a123a509a19
KUBE_APISERVER="https://${master_ip}:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 创建kube-proxy kubeconfig文件
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

**更新node结点kubeconfig文件**

```
scp /etc/kubernetes/*.kubeconfig root@node:/etc/kubernetes
systemctl restart kubelet kube-proxy
```

## kubernetes node部署

*更新kube-proxy、kubelet配置文件, hostname-override是当前机器主机名*

*kubelet配置文件: node-labels=node-role.kubernetes.io/master 更新为node-labels=node-role.kubernetes.io/node*

```
scp /etc/kubernetes/ssl/* root@node:/etc/kubernetes/ssl
scp /etc/kubernetes/*.kubeconfig root@node:/etc/kubernetes
scp /etc/kubernetes/kube-proxy.conf root@node:/etc/kubernetes
scp /etc/kubernetes/kubelet.conf root@node:/etc/kubernetes
scp /usr/bin/kube-proxy root@node:/usr/bin
scp /usr/bin/kubelet root@node:/usr/bin
scp /lib/systemd/system/kubelet.service root@node:/lib/systemd/system
scp /lib/systemd/system/kube-proxy.service root@node:/lib/systemd/system
```

**启动服务**

```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl enable kube-proxy
systemctl start kube-proxy
```

## node结点加入集群

```
kubectl get csr
# 手动接收认证
kubectl certificate approve $NAME
```