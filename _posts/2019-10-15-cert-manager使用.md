---
layout: post
title: cert-manager的安装与Ingress使用
subtitle: 'cert-manager'
date: 2019-10-15
categories: blog
tags: [cert-manager,Ingress,dns01,http01]
---
# cert-manager的安装与Ingress使用
* 安装cert-manager
   这里使用yaml安装

   ```bash
   kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.11.0/cert-manager.yaml
   ```
## http01 模式

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
      email: w@qq.com
      privateKeySecretRef:
        name: letsencrypt-prod
      solvers:
      - selector: {}
        http01:
         ingress: {}
  ```
## dns01 模式
 - 支持的DNS01提供商
 - ACMEDNS
 - Akamai
 - AzureDNS
 - CloudFlare
 - Google
 - Route53 
 - DigitalOcean
 - RFC2136
 - 详情查看[https://cert-manager.io/docs/configuration/acme/dns01/] (https://cert-manager.io/docs/configuration/acme/dns01/)

  ```bash
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: example-issuer-account-key
    solvers:
    - dns01:
        clouddns:
          project: my-project
          serviceAccountSecretRef:
            name: prod-clouddns-svc-acct-secret
            key: service-account.json
  ```
- 如果你使用的dns供应商不在列表内，那就要通过Webhook来提供支持。
  我这里使用阿里云dns解析`https://github.com/pragkent/alidns-webhook`
  由于0.11版本变更了apiversion需要注意
  
```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alidns-webhook
  namespace: cert-manager
  labels:
    app: alidns-webhook

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: alidns-webhook
  namespace: cert-manager
  labels:
    app: alidns-webhook
rules:
  - apiGroups:
      - ''
    resources:
      - 'secrets'
    verbs:
      - 'get'

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: alidns-webhook
  namespace: cert-manager
  labels:
    app: alidns-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: alidns-webhook
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: alidns-webhook
    namespace: cert-manager

---
# Grant the webhook permission to read the ConfigMap containing the Kubernetes
# apiserver's requestheader-ca-certificate.
# This ConfigMap is automatically created by the Kubernetes apiserver.
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: alidns-webhook:webhook-authentication-reader
  namespace: kube-system
  labels:
    app: alidns-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: alidns-webhook
    namespace: cert-manager
---
# apiserver gets the auth-delegator role to delegate auth decisions to
# the core apiserver
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: alidns-webhook:auth-delegator
  namespace: cert-manager
  labels:
    app: alidns-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: alidns-webhook
    namespace: cert-manager
---
# Grant cert-manager permission to validate using our apiserver
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: alidns-webhook:domain-solver
  labels:
    app: alidns-webhook
rules:
  - apiGroups:
      - acme.yourcompany.com
    resources:
      - '*'
    verbs:
      - 'create'
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: alidns-webhook:domain-solver
  labels:
    app: alidns-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: alidns-webhook:domain-solver
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: cert-manager
    namespace: cert-manager

---
# Source: alidns-webhook/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: alidns-webhook
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: https
      protocol: TCP
      name: https
  selector:
    app: alidns-webhook

---
# Source: alidns-webhook/templates/deployment.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: alidns-webhook
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  replicas: 
  selector:
    matchLabels:
      app: alidns-webhook
  template:
    metadata:
      labels:
        app: alidns-webhook
    spec:
      serviceAccountName: alidns-webhook
      containers:
        - name: alidns-webhook
          image: pragkent/alidns-webhook:0.1.0
          imagePullPolicy: IfNotPresent
          args:
            - --tls-cert-file=/tls/tls.crt
            - --tls-private-key-file=/tls/tls.key
          env:
            - name: GROUP_NAME
              value: "acme.yourcompany.com"
          ports:
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /healthz
              port: https
          readinessProbe:
            httpGet:
              scheme: HTTPS
              path: /healthz
              port: https
          volumeMounts:
            - name: certs
              mountPath: /tls
              readOnly: true
          resources:
            {}
            
      volumes:
        - name: certs
          secret:
            secretName: alidns-webhook-webhook-tls

---
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.acme.yourcompany.com
  labels:
    app: alidns-webhook
  annotations:
    cert-manager.io/inject-ca-from: "cert-manager/alidns-webhook-webhook-tls"
spec:
  group: acme.yourcompany.com
  groupPriorityMinimum: 1000
  versionPriority: 15
  service:
    name: alidns-webhook
    namespace: cert-manager
  version: v1alpha1

---
# Create a selfsigned Issuer, in order to create a root CA certificate for
# signing webhook serving certificates
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: alidns-webhook-selfsign
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  selfSigned: {}

---

# Generate a CA Certificate used to sign certificates for the webhook
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: alidns-webhook-ca
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  secretName: alidns-webhook-ca
  duration: 43800h # 5y
  issuerRef:
    name: alidns-webhook-selfsign
  commonName: "ca.alidns-webhook.cert-manager"
  isCA: true

---

# Create an Issuer that uses the above generated CA certificate to issue certs
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: alidns-webhook-ca
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  ca:
    secretName: alidns-webhook-ca

---

# Finally, generate a serving certificate for the webhook to use
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: alidns-webhook-webhook-tls
  namespace: cert-manager
  labels:
    app: alidns-webhook
spec:
  secretName: alidns-webhook-webhook-tls
  duration: 8760h # 1y
  issuerRef:
    name: alidns-webhook-ca
  dnsNames:
  - alidns-webhook
  - alidns-webhook.default
  - alidns-webhook.default.svc
```
  
- Issuer示例 这是测试用server

```bash
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: w@qq.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.yourcompany.com
          solverName: alidns
          config:
            region: ""
            accessKeySecretRef:
              name: alidns-secret
              key: access-key
            secretKeySecretRef:
              name: alidns-secret
              key: secret-key
```
  
- aliyun密钥key base64加密过的

```bash
apiVersion: v1
kind: Secret
metadata:
  name: alidns-secret
data:
  access-key: xxxxxxxxxxxxxxxxxxxxx
  secret-key: xxxxxxxxxxxxxxxxxxxxxxx
```

- 颁发证书

* dns01支持通配符

```bash
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: wangyp.ww
spec:
  secretName: wangyp.ww-tls
  commonName: wangyp.ww
  dnsNames:
  - wangyp.ww
  - "*.wangyp.ww"
  issuerRef:
    name: letsencrypt-staging
    kind: Issuer
```

* http01不支持通配符

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

  

