---
title: 通过kube-apiserver访问kube-dashboard
date: 2020-05-16 13:08:40
tags: kubernetes
---

# 制作证书

<!--more-->


**client-certificate-data**

```
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
```


**client-key-data**

```
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
```


**生成ps12文件**

```
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```


> tips: 浏览器导入证书之后需要重启