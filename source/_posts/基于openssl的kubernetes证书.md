---
title: 基于openssl的kubernetes证书
date: 2020-02-11 12:49:10
tags: kubernetes, openssl
---

# kube-apiserver的CA证书

```
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=<master_hostname>" \
    -days 5000 -out ca.crt
openssl genrsa -out server.key 2048
```

<!--more-->

*master_ssl.cnf*

```
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = <master_hostname>
IP.1 = <master_ip>
IP.3 = <kubectl get svc | grep kubernetes | awk '{print $3}'>集群IP
```

**基于master_ssl.conf创建server.csr和server.crt文件**

```
openssl req -new -key server.key -subj "/CN=<master_hostname>" -config \
master_ssl.cnf -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-days 5000 -extensions v3_req -extfile master_ssl.cnf -out server.crt
```


**kube-apiserver启动参数**

```
--client-ca-file=<ca_dir>/ca.crt
--tls-private-key-file=<ca_dir>/server.key
--tls-cert-file=<ca_dir>/server.crt
--insecure-port=0
--secure-port=6443
```


# kube-controller-manager证书

```
openssl genrsa -out cs_client.key 2048
openssl req -new -key ca.key -subj "/CN=<master_hostname>" -out cs_client.csr
openssl x509 -req -in cs_client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
-out cs_client.crt -days 5000
```

*controller-manager.kubeconfig*

```
apiVersion: v1
kind: Config
users:
- name: controllermanager
  user:
    client-certificate: <ca_dir>/cs_client.crt
    client-key: <ca_dir>/cs_client.key
clusters:
- name: local
  cluster:
    certificate-authority: <ca_dir>/ca.crt
    server: https://<apiserver_ip>:6443
contexts:
- context:
    cluster: local
    user: controllermanager
  name: my-context
current-context: my-context
```

*kube-controller-manger服务的启动参数*

```
--service-account-key-file=<ca_dir>/server.key
--root-ca-file=<ca_dir>/ca.crt
--kubeconfig=controller-manager.kubeconfig
```

# kube-scheduler

*启动参数*

```
--kubeconfig=controller-manager.kubeconfig
```


# kubelet证书

```
openssl genrsa -out kubelet_client.key 2048
openssl req -new -key kubelet_client.key -subj "/CN=<node_hostname>" -out \
kubele_client.csr
openssl x509 -req -in kubelet_client.csr -CA ca.crt -CAkey ca.key \
-CAcreateserial -out kubelet_client.cet -days 5000
```

*kubelet.kubeconfig*

```
apiVersion: v1
kind: Config
users:
- name: kubelet
  user: 
    client-certificate: <ca_dir>/kubelet_client.crt
    client-key: <ca_dir>/kubelet_client.key
clusters:
- name: local
  cluster:
    certificate-authority: <ca_dir>/ca.crt
    server: https://<api_server>:6443
contexts:
- context:
    cluster: local
    user: kubelet
  name: my-context
current-context: my-context
```

**kubelet服务启动参数**

```
--kubeconfig=kubelet.kubeconfig
```

# kube-proxy


```
--kubeconfig=kubelet.kubeconfig
```


# kubectl


*安全访问apiserver*

```
kubectl --server=https://<apiserver>:6443 \
--certificate-authority=<ca_dir>/ca.crt \
--client-certificate=<ca_dir>/cs_client.crt \
--clinet-key=<ca_dir>/cs_client.key
```