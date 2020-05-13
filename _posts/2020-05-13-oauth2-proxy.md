---
layout: k8s
title: oauth2认证
date: 2020-05-13
categories: blog
tags: [Kubernetes-dashboard,oauth2-proxy]
---

# Kubernetes-dashboard的身份认证

## 前言

开放公网访问Kubernetes仪表板的最佳方法是使用身份验证代理。这是位于服务前的代理，仅在用户通过身份验证后才允许流量通过。这里基于开源[oauth2-proxy的身份验证](https://github.com/oauth2-proxy/oauth2-proxy) 。它与Kubernetes仪表板一起可以很好地工作。

这里我演示的是已经安装配置好的Kubernetes-dashboard基础上添加oauth2-proxy认证

在此示例中，我们还将使用GitHub作为我们的身份验证提供程序。oauth2_proxy支持多个供应商。有关详细信息，请参阅该官方文档。

流程图，我们将看到如下所示的内容。请求流量将遵循粗体箭头。Ingress将使用“cert-manager”获取将在互联网上使用的证书。在oauth2_proxy随后将与[GitHub](https://github.com) 进行身份验证（将用户访问重定向到GitHub上）。认证通过后，流量将传递到仪表板，并将使用API​​服务器。

![流程图](https://wangyp.cf/assets/img/1_My-azKvnd_VgJsbRKWPlNw.png)

## 建立GitHub oauth2应用
oauth2_proxy支持添加github个人或组织
登录 [https://github.com/settings/developers](https://github.com/settings/developers) 并创建一个新的oauth应用程序。记录id，Secret信息。正确的关键是回调URL。
将域名设置为 **自己的域名**

![](https://wangyp.cf/assets/img/20200513154536.png)

## 启动oauth2-proxy实例

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  labels:
    k8s-app: oauth2-proxy
  name: oauth2-proxy
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: oauth2-proxy
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: oauth2-proxy
    spec:
      containers:
      - args:
        - --provider=github
        - --cookie-expire=2h
        - --github-org=cnyappam
        - --email-domain=*
        - --http-address=0.0.0.0:4180
        - --set-authorization-header=true
        - --whitelist-domain=.k8s.wangyp.win
        - --cookie-domain=.k8s.wangyp.win
        env:
        - name: OAUTH2_PROXY_CLIENT_ID
          value: xxxxxxx
        - name: OAUTH2_PROXY_CLIENT_SECRET
          value: xxxxxx
        - name: OAUTH2_PROXY_COOKIE_SECRET
          value: hRyFB1hSCQtuA1NmWcUKFg==
        image: registry.yappam.com/pusher/oauth2-proxy:v5.1.1
        imagePullPolicy: Always
        name: oauth2-proxy
        ports:
        - containerPort: 4180
          protocol: TCP
		  
---
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    k8s-app: oauth2-proxy
  name: oauth2-proxy
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 4180
    protocol: TCP
    targetPort: 4180
  selector:
    k8s-app: oauth2-proxy
  type: ClusterIP
  

```

## 配置Ingress

这里使用Authorization header方式认证Dashboard [Dashboard认证](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md#bearer-token)

```bash
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/auth-response-headers: Authorization
    nginx.ingress.kubernetes.io/auth-signin: https://$host/oauth2/start?rd=https://$host$request_uri$is_args$args
    nginx.ingress.kubernetes.io/auth-url: https://$host/oauth2/auth
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    #nginx.ingress.kubernetes.io/auth-snippet: |
    #  proxy_set_header Authorization "Bearer $cookie_jwt_token";
    nginx.ingress.kubernetes.io/configuration-snippet: |
	  #token
      proxy_set_header 'Authorization' 'Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWdsNmJmIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjYWM2MGYyOC1hZjE4LTQyMTItODViMi1jODNmOWY0MDZjZTgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.uwmHPC3M9OpcdG38VH9vzJShIPUM4YoQ-WIxSvMw4jT6cF28gnyNiln_F8ZDN3FivVK2nLM9JXbjxEjwbEJZ3adsvJ-qefnF3YcOzL68tRLEygFos6XyiGa6NRt3jx5zGq2pi2LC8dGkbYv7tbZ39nHE0JO1Nf6l8_W0oAuWRzvfjUtnXZWX7tmWuZeNY9AH2M61Pfx8ndeZtuAnYho-yFxMd9uingJ_Eie4VA0nIf2S_CLZLbV4NupJcIN4X1QxlLMpFzuv35TLlPM0fZpbEteXbM-br6uuRaWj96UIx_VRDqDcwzJfYKJxW_7hvsD-223AL9IapkY6mEJ-S1jSXQ';
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  rules:
  - host: kubernetes-dashboard.k8s.wangyp.win
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
        path: /
  tls:
  - hosts:
    - kubernetes-dashboard.k8s.wangyp.win
    secretName: dashboard-ingress-secret

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
  name: oauth2-proxy
  namespace: kube-system
spec:
  rules:
  - host: kubernetes-dashboard.k8s.wangyp.win
    http:
      paths:
      - backend:
          serviceName: oauth2-proxy
          servicePort: 4180
        path: /oauth2
  tls:
  - hosts:
    - kubernetes-dashboard.k8s.wangyp.win
    secretName: dashboard-ingress-secret
```

## 访问

浏览器访问kubernetes-dashboard.k8s.wangyp.win会跳转到github认证

![renzheng](https://wangyp.cf/assets/img/20200513162035.png)

认证成功后跳转dashboard

![renzheng](https://wangyp.cf/assets/img/20200513162343.png)
