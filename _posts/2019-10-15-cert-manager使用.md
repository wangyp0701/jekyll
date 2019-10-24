---
layout: post
title: cert-manager的安装与Ingress使用
date: 2019-10.15
categories: blog
tags: [cert-manager,Ingress]
description: 文章金句。
---
# cert-manager的安装与Ingress使用
* 安装cert-manager
   这里使用yaml安装

   ```bash
   kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
   ```

* 配置ClusterIssuer

  [ClusterIssuer](https://cert-manager.readthedocs.io/en/latest/reference/clusterissuers.html)是[Issuer](https://cert-manager.readthedocs.io/en/latest/reference/issuers.html)的集群范围版本。[证书](https://cert-manager.readthedocs.io/en/latest/reference/certificates.html)资源可以在任何命名空间中引用它。
  
  ```bash
  apiVersion: cert-manager.io/v1alpha2
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-prod
  spec:
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory
      email: wangyunpeng@yappam.com
      privateKeySecretRef:
        name: letsencrypt-prod
      solvers:
      - selector: {}
        http01:
         ingress: {}
  ```
  
* 配置Certificate

  ```bash
  apiVersion: cert-manager.io/v1alpha2
  kind: Certificate
  metadata:
    name: my-certificate
    namespace: nginx
  spec:
    renewBefore: 12h
    secretName: my-certificate-secret
    issuerRef:
      name: letsencrypt-prod
      kind: ClusterIssuer
    commonName: yy.wangyp.win
    dnsNames:
    - "nginx.yy.wangyp.win"
  ```

* 配置Ingress

  ```bash
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: nginx
    namespace: nginx
    annotations:
      #添加一个注释，指示要使用的发行者。
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
  spec:
    tls:
    - hosts:
      - "nginx.yy.wangyp.win"
      secretName: my-certificate-secret
    rules:
    - host: "nginx.yy.wangyp.win"
      http:
        paths:
        - path: /
          backend:
            serviceName: nginx
            servicePort: 80
  ```

  

