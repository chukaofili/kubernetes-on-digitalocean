# Add Persistent Volume Support Using DigitalOcean Block Storage - Part II

This is 2nd part in the [**$65 Kubernetes Cluster on DigitalOcean** series](./README.md), you can goto [Part I](./Part-I.md) to read on how to setup your cluster if you haven't done so yet.

There's also a [video tutorial here](https://youtu.be/OrO6-4AWvD0) for those who prefer to watch instead of read.



## Introduction 

By default, when you setup a kubernetes cluster on digitalocean manually, there isn't any persistent volume support even though digitalocean has block storage

Our aim is to enable **persistent volume** support backed by digitaloceans block storage using a storage provisioner plugin. This tutorial assumes you have a running kubernetes cluster setup on digitalocean using CoreOS (setup might vary for other operating systems) with RBAC enabled (usually enabled by default with versions 1.9 and above).

You'll need a digitalocean access token, get one from [your account here](https://cloud.digitalocean.com/settings/api/tokens).



## Step 1: Configure Access Token

Base64 encode your digitalocean access token using the command line (macOs):

```shell
base64 <<< [digital-ocean-token-here]
```

It should output and encoded string, something like this:

```shell
W2RpZ2l0YWwtb2NlYW4tdG9rZW4taGVyZV0K
```

Insert the encoded string into the following yaml file and save it your system as  `digitalocean-secret.yml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: digitalocean
  namespace: kube-system
data:
  access-token: [base64-encoded-string-goes-here]
```

And finally create the secret using the command:

```shell
kubectl create -f digitalocean-secret.yml
```



## Step 2: Update kubelet service with volume plugin directory

We'll need to create the volume plugin directory and tell the kubelet service where the directory lives, this has to be done on the kubenetes master. Save this script as `blockstorage-pv.sh`

```bash
#!/bin/bash

sudo su
mkdir -p /etc/kubernetes/kubelet-plugins/volume
sed -i -e 's#\(KUBELET_EXTRA_ARGS=\)#\1--volume-plugin-dir=/etc/kubernetes/kubelet-plugins/volume #' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet
```

And run the following command:

```shell
ssh core@[kubernetes-master-ip-goes-here] "bash -s" < ./blockstorage-pv.sh
```

If everything goes well, it should exit with out any errors.



## Step 3: Update kube-controller-manager

Next we'll need to update the kube-controller manager with the right path to ssl certs, as the defaults don't exist, and we'll need point it to the default volume plugin directory. Ssh into your kubernetes master with `ssh core@[kubernetes-master-ip-goes-here] ` and update the following file `/etc/kubernetes/manifests/kube-controller-manager.yaml` using the **root user**:

Under `spec.containers.command` **add** the following:

```yaml
- --flex-volume-plugin-dir=/etc/kubernetes/kubelet-plugins/volume
```

Under `spec.containers.volumeMounts` **add** the following:

```yaml
    - mountPath: /etc/kubernetes/kubelet-plugins/volume
      name: flexvolume-mount
      readOnly: true
```

Under `spec.volumes` **update** the following:

```yaml
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
```

with this yaml (this will update the ssl certs to the right path):

```yaml
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: ca-certs
```

And then **add** the flex volume-mount:

```yaml
  - hostPath:
      path: /etc/kubernetes/kubelet-plugins/volume
      type: DirectoryOrCreate
    name: flexvolume-mount
```

Save the file and finally restart the sublet service with `systemctl restart kubelet`



## Step 4: Deploy the digitalocean storage provisioner plugin

#### Deploy RBAC rules

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: digitalocean-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: digitalocean-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: digitalocean-provisioner
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: digitalocean-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: digitalocean-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: digitalocean-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: digitalocean-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: digitalocean-provisioner
subjects:
- kind: ServiceAccount
  name: digitalocean-provisioner
```

Save the **rbac rules** as **`digitalocean-flexplugin-rbac.yml` **and create the rules using the following:

```shell
kubectl create -f digitalocean-flexplugin-rbac.yml
```



#### Deploy digitalocean provisioner

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    app: digitalocean-provisioner
  name: digitalocean-provisioner
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: digitalocean-provisioner
  template:
    metadata:
      labels:
        app: digitalocean-provisioner
    spec:
      containers:
      - env:
        - name: DIGITALOCEAN_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              key: access-token
              name: digitalocean
        image: quay.io/external_storage/digitalocean-provisioner
        imagePullPolicy: Always
        name: digitalocean-provisioner
        resources:
          limits:
            cpu: 50m
            memory: 64Mi
          requests:
            cpu: 50m
            memory: 64Mi
      serviceAccount: digitalocean-provisioner
```

Save the **provisioner** as **`digitalocean-provisioner.yml`** and deploy using the following:

```shell
kubectl create -f digitalocean-provisioner.yml
```



#### Deploy the digitalocean flexplugin

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  labels:
    app: digitalocean-flexplugin-deploy
  name: digitalocean-flexplugin-deploy
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: digitalocean-flexplugin-deploy
  template:
    metadata:
      labels:
        app: digitalocean-flexplugin-deploy
      name: digitalocean-flexplugin-deploy
    spec:
      containers:
      - env:
        - name: DIGITALOCEAN_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              key: access-token
              name: digitalocean
        image: quay.io/external_storage/digitalocean-flexplugin
        imagePullPolicy: Always
        name: flex-deploy
        resources:
          limits:
            cpu: 25m
            memory: 32Mi
          requests:
            cpu: 25m
            memory: 32Mi
        volumeMounts:
        - mountPath: /flexmnt
          name: flexvolume-mount
      tolerations:
      - operator: Exists
      volumes:
      - hostPath:
          path: /etc/kubernetes/kubelet-plugins/volume
        name: flexvolume-mount
```

Save the **flexplugin** as **`digitalocean-flexplugin-deploy.yml`** and deploy using the following:

```shell
kubectl create -f digitalocean-flexplugin-deploy.yml
```



#### Deploy the storage class

>  **Important!**: Change the zone on **Line 8** below to the **same region** as your cluster.

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: do-lon1
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
parameters:
  zone: [region-goes-here]
provisioner: external/digitalocean
```

Save the **storage class** as **`ditigalocean-sc.yml`** and deploy using the following:

```shell
kubectl create -f ditigalocean-sc.yml
```



## Step 5

Let's deploy a sample application which will utilise a **persistent volume** to make sure our deployment is working.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi
  storageClassName: do-lon1
---
apiVersion: v1
kind: Pod
metadata:
  name: busy-pod
spec:
  containers:
  - command:
    - /bin/sleep
    - 7d
    image: busybox
    name: busy-container1
    volumeMounts:
    - mountPath: /mnt/pv-1
      name: vol1
  volumes:
  - name: vol1
    persistentVolumeClaim:
      claimName: pv1
```

 Save the **deployment** as **`ditigalocean-pv-example.yml`** and deploy using the following:

```shell
kubectl create -f ditigalocean-pv-example.yml
```



To check If your deployment succeeds, goto your digitalocean account under **Droplets > Volumes** you should see a **1Mb volume** provisioned and attached to one of your nodes. If this is the case, you have successfully added persistent volume support to your kubernetes cluster. Yay!!!



## Conclusion

Next in our series, we'll install and enable our **kubernetes dashboard**! But still to come, installing helm & automatic ssl certificates backed by letsencrypt. Stay tuned.



I hope this helps.