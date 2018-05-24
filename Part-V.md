# Automate TLS/SSL Certificate Issuance & Renewal - Part V

![Title Image](http://gdurl.com/K2Ra)

This is the 5th and final part in our [**$65 Kubernetes Cluster on DigitalOcean**](./README.md) series, you can goto [Part I](./Part-I.md) to read on how to setup your cluster if you haven't done so yet.

There's also a [video tutorial here](https://youtu.be/aB0TagEzTAw) for those who prefer to watch instead of read.



## Introduction

Transport Layer Security (**TLS**) – and its predecessor, Secure Sockets Layer (**SSL**) are cryptographic protocols that provide communications security over a computer network. It activates the padlock and the **https** protocol and allows secure connections from a web server to a browser.

Traditionally these certificates can cost anywhere from `$30 to $500` depending on the level of encryption and validation required. But for most websites a simple and basic TLS/SSL certificate should do and [letsencrypt.org](https://www.letsencrypt.org) offers them for **free**!

What we will focus on today is how to automate our kubernetes cluster in issuing TLS/SSL certificates from the [letsencrypt.org](https://www.letsencrypt.org) api using a tool called **cert-manager**.



## Step 1 - Install Cert-Manager 

We'll be using helm to install **cert-manager,** if you don't have helm installed you can read up [here](./Part-IV.md) to quickly have it installed. Also the nginx-ingress should already be installed fully configured, you can read up [here](./Part-I.md#configure-nginx-ingress) to quickly install it. 

The below will install cert-manager to the kube-system namespace.

```shell
helm install --name cert-manager --namespace kube-system stable/cert-manager
```



## Step 2 - Configure Certificate Issuer

Before cert-manager can vend certificates, it needs a backing certifictate issuer, we be using [letsencrypt.org](https://www.letsencrypt.org)  for certificate issuance. 

```yaml

apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging.api.letsencrypt.org/directory
    email: [your-email-goes-here]
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v01.api.letsencrypt.org/directory
    email: [your-email-goes-here]
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}
```

> **Note**: Replace **Lines 8 & 20** with your email address, this is needed to generate your key pair for issuing certificates from letsencrypt.

Save this yaml file as `cert-manager-cluster-issuer.yaml`

```shell
kubectl apply -f ./cert-manager-cluster-issuer.yaml
```



## Step 3 - Example Certificate

Now everything should be configured correctly. Let's test it out by creating a sample deployment. 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: echoserver
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: echoserver
  namespace: echoserver
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
      - image: gcr.io/google_containers/echoserver:1.0
        imagePullPolicy: Always
        name: echoserver
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echoserver
  namespace: echoserver
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: echoserver
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echoserver
  namespace: echoserver
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - echo.[your-domain-goes-here]
    secretName: echoserver-tls-prod
  rules:
  - host: echo.[your-domain-goes-here]
    http:
      paths:
      - path: /
        backend:
          serviceName: echoserver
          servicePort: 80
```

> **Note**: Replace **Lines 49 & 52** with your domain name (this domain should already point to your kubernetes cluster), this is needed to generate your key pair for issuing certificates from letsencrypt.

Save file as `echo-server-tls.yaml`

```shell
kubectl apply -f ./echo-server-tls.yaml
```

Goto your domain at **echo.[your-domain-goes-here]** and you should see that it has been configured with a TLS/SSL certificate.



## Conclusion

There is more information in the [official docs](https://cert-manager.readthedocs.io) about configuring other **Issuers** and also other annotations that can be used in your ingress manifests.



I hope this helps.



***

[<< Previous](Part-IV.md)