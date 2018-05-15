# Kubernetes on Digital Ocean (CoreOS)

![kubernetes_degitalocean](./assets/kubernetes_digitalocean-compressor.jpg)

This is a multipart tutorial and walkthrough on setting up kubernetes on DigitalOcean's droplets using CoreOS. It's mostly manual setup until DigitalOcean releases thier managed kubernetes service [here](https://www.digitalocean.com/products/kubernetes/).

## Thing's you will need.

1. You'll need a Digital Ocean account, if you don't have you can get one with a free $10 credit [here](https://m.do.co/c/abb066bf4bc9).

2. You'll also need to install `kubectl`

   * If you are on macOS and using [Homebrew](https://brew.sh/) package manager, you can install with: 

     ```bash
     brew install kubectl
     ```

   * On Linux

     ```bash
     curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
     chmod +x ./kubectl
     sudo mv ./kubectl /usr/local/bin/kubectl
     ```

   * On Windows

     ```shell
     curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/windows/amd64/kubectl.exe
     ```
     > Add the binary to your PATH.



## Kubernetes Master

Create a droplet using the current stable version of CoreOS. Minimum recommended droplet spec: 2GB Ram, 1 vCPU. Choose your preferred region, add your ssh-key (you won't be able to create the droplet without it), enter your preferred hostname and finally add the tag `k8s-master`.

> Note: Do not add block storage and remember the region used, you'll need it for the worker nodes later on. 

<script src="https://gist.github.com/chukaofili/06844c424e9f7c6ee387a70b90f3b8ac.js"></script>