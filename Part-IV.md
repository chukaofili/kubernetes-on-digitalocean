# Installing Helm Charts - Part IV

![Title Image](http://gdurl.com/N8aM)

This is the 4th part in our [**$65 Kubernetes Cluster on DigitalOcean**](./README.md) series, you can goto [Part I](./Part-I.md) to read on how to setup your cluster if you haven't done so yet.

There's also a [video tutorial here](https://youtu.be/aB0TagEzTAw) for those who prefer to watch instead of read.



## Introduction

Helm Charts is the ultimate package manager for kubernetes apps. Imagine as homebrew is to macOS and yarn/npm is to node, this is what Helm is to kubernetes. 

Helm Charts helps to automate the installation of applications to our kubernetes cluster and can quickly help us bootstrap, configure and install repeatable production ready apps .

Let's say we wanted to install a **mongodb** deployment, typically we would have to prepare our manifests files, which would normally include a **deployment,** **service,** **configmaps,** **secrets** and maybe **ingress** config. But with helm, all we would have to do would be run something like `helm install stable/mongodb` and helm would install and configure **mongodb** on our behalf. Fun isn't it?? :)



TL;DR;

## Step 1 - Installation

Installing helm is as easy as `brew install kubernetes-helm` (macOS/homebrew). For other platforms, check out the [installation guide](https://docs.helm.sh/using_helm/#installing-helm). This will install the **cli tool.**



## Step 2 - Initialisation

Once the cli tool has been installed, we will need to the install **Tiller server** to our cluster and also the create the service account that will be used. 



```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Save file as `helm-rbac-config.yaml`

```shell
kubectl apply -f ./helm-rbac-config.yaml
```

Once the service account has been created, run the following command:

> **Note:** *Make sure you are in the right kubernetes **cluster-context** before you proceed*

```shell
helm init --service-account tiller
```



## Step 3 - Example MongoDB Installation

The easiest way to install an application (also referred to as a chart), is by using the official **stable** channel.

To install **mongodb** and give the release a name **simpledb**, run the following:

```shell
helm install --name simpledb stable/mongodb
```

Once a chart is installed, a new release is created so as to enable us reuse the same chart over and over in our cluster. To get more information about our **simpledb** mongodb deployment run;

```shell
helm status simpledb
```



## Conclusion

So there we have it, our example **mongodb** installation should be up and running. You can find out more about using helm from the official docs [here](https://docs.helm.sh/using_helm).



I hope this was helpful.



------

[<< Previous](Part-III.md)