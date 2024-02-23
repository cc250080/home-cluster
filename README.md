# flux-cluster
Welcome to my **really** highly opinionated Repo for my GitOps home-lab. In this repo I install a basic Vanilla [Kubernetes](https://kubernetes.io/docs/home/) and I use [Flux](https://fluxcd.io/flux/) to manage its state.

## 👋 Introduction
My personal Goal with this project is to have an easy and elegant way to manage applications that I want to run in my kubernetes home-lab while at the same time use it to keep learning the intrinsicacies of Kubernetes and GitOps. That is why I took some choices like for example installing Vanilla Kubernetes by hand. You won't find here *Ansible* playbooks or other automatisms to install Kubernetes, Flux and its tools. For now I choose to install everything by hand and learn and interiorize during the process. It is my goal to take out any abstraction on top of the basic Kubernetes components while at the same time enjoying a useful GitOps installation.

## ✨ Features

- Documented **manual** Installation of Kubernetes v1.29 mono-node cluster in Debian 12 *BookWorm*
- Opinionated implementation of Flux with [strong community support](https://github.com/onedr0p/flux-cluster-template#-support)
- Encrypted secrets thanks to [SOPS](https://github.com/getsops/sops) and [Age](https://github.com/FiloSottile/age)
- Web application firewall thanks to [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps)
- SSL certificates thanks to [Cloudflare](https://cloudflare.com) and [cert-manager](https://cert-manager.io)
- Next-gen networking thanks to [Cilium](https://cilium.io/)
- A [Renovate](https://www.mend.io/renovate)-ready repository
- Integrated [GitHub Actions](https://github.com/features/actions)

... and more!

## 📝 Pre-start checklist

Before getting started everything below must be taken into consideration, you must...

- [ ] run the cluster on bare metal machines or VMs within your home network &mdash; **this is NOT designed for cloud environments**.
- [ ] have Debian 12 freshly installed on 1 or more AMD64/ARM64 bare metal machines or VMs. Each machine will be either a **control node** or a **worker node** in your cluster.
- [ ] give your nodes unrestricted internet access &mdash; **air-gapped environments won't work**.
- [ ] have a domain you can manage on Cloudflare.
- [ ] be willing to commit encrypted secrets to a public GitHub repository.
- [ ] have a DNS server that supports split DNS (e.g. [AdGuardHome](https://github.com/AdguardTeam/AdGuardHome) or<Down> Pi-Hole) deployed somewhere outside your cluster **ON** your home network.

## 💻 Machine Preparation

### System Requirements

According to the Official Documentation the [requirements](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#before-you-begin) for kubernetes are just 2 Cores and 2 Gb of Ram to install a Control-Plane node, but obviously only with this we would be leaving little room left for our apps.

📍 For my home-lab I will only be installing one Control-Plane node, so to run apps in it I will remove the Kubernetes *Taint* that block worker nodes from running normal Pods.

📍 I Choose Debian Stable because IMO is the best hassle-free, community-driven and stability and security focused distribution.

### 🌀 Debian Installation

Perform a basic server installation **without Swap partition or SwapFile**, Kubernetes and Swap aren't friends yet.

After Debian is installed:

1. [Post install] Enable sudo for your non-root user

    ```sh
    su -
    apt update
    apt install -y sudo
    usermod -aG sudo ${username}
    echo "${username} ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/${username}
    exit
    newgrp sudo
    sudo apt update
    ```

2. [Post install] Add SSH keys (or use `ssh-copy-id` on the client that is connecting)

    📍 _First make sure your ssh keys are up-to-date and added to your github account as [instructed](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)._

    ```sh
    mkdir -m 700 ~/.ssh
    sudo apt install -y curl
    curl https://github.com/${github_username}.keys > ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    ```

3. [Containerd] Configure Kernel modules and pre-requisites for containerd

    ```sh
    cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
    overlay
    br_netfilter
    EOF
    ```

    ```sh
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

    ```sh
    cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
    ```

    ```sh
    sudo sysctl --system
    ```

4. [Containerd] Install Containerd

    ```sh
    sudo apt-get update
    sudo apt-get install containerd
    ```

5. [Containerd] Set the configuration file for Containerd:

    ```sh
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    ```

6. [Containerd] Set cgroup driver to systemd:


    ```sh
    sudo vim /etc/containerd/config.toml
    ```

    ```sh
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
          base_runtime_spec = ""

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
          SystemdCgroup = true
    ```

Edit the containerd configuration file and ensure that the option SystemdCgroup is set to "true".

After that restart containerd and ensure everything is up and running properly.


    sudo systemctl restart containerd.service
    sudo systemctl status containerd.service


### 🔧 Install Kubernetes with KubeAdm and without KubeProxy

When we already have a CRI installed(containerd) and starting properly using systemd it is the time to install K8s, for this guide we want to provision a Kubernetes cluster without *kube-proxy*, and to use *Cilium* to fully replace it. For simplicity, we will use kubeadm to bootstrap the cluster. For help with installing *kubeadm* and for more provisioning options please refer to [the official Kubeadm documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

Then, after installing kubeadm, kubectl and kubelet and perform the **kubeadm init** we can ensure everything is *almost* up and ready in our cluster:

```sh
>  sudo kubeadm init --skip-phases=addon/kube-proxy
```

```sh
~ on  main [!?] 
❯ k get nodes
NAME       STATUS   ROLES           AGE   VERSION
bacterio   UnReady    control-plane   5d    v1.29.3
```
In order to get a Ready state, we still have to install our Pod Network Add-on or **CNI**.

### 🌐 Installing the Container Network Interface (CNI) add-on

In order to install **Cilium** it is enough to follow the [Cillium Quick Installation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/), we will need the *Cilium CLI* tool and *Cilium* installed in our Cluster.

And after that the Cluster will become Ready.

```sh
~ on  main [!?] 
❯ k get nodes
NAME       STATUS   ROLES           AGE   VERSION
bacterio   Ready    control-plane   5d    v1.29.3
```
### Untaint the Cluster to be able to Run Pods

Finally, we untaint the *Control-Plane* node to allow it to run workloads and test that it can in fact run a *Pod*.

```sh
kubectl taint nodes bacterio node-role.kubernetes.io/control-plane:NoSchedule-
```

```sh
kubectl run testpod --image=nginx
kubectl get pods
```
## 🤖 Installing Flux

[Flux](https://fluxcd.io/) is a set of continuous and progressive delivery solutions for Kubernetes that are open and extensible. In a more plain language Flux is a tool for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), that way we can use GIT a source of truth and use it to interact with our Cluster.

### Install the Flux CLI

The **Flux** command-line interface (CLI) is used to bootstrap and interact with Flux.

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

And [here](https://fluxcd.io/flux/cmd/flux_completion_bash/) you can find instructions to add *bash* autocompletion features to the shell.

### Export your credentials

Export your GitHub personal access token and username:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

The *GITHUB_TOKEN* in use here is **PAT**(Personal Access Token)

### Check your Kubernetes Cluster

We can use the Flux CLI to run a pre-flight check on our Cluster and see if we fulfill all the basic requirements to install **Flux**.

```sh
flux check --pre
```
Which should produce an output like:
```sh
► checking prerequisites
✔ kubernetes 1.29.2 >=1.25.0
✔ prerequisites checks passed
```

### Installing Flux onto the Cluster

For information on how to bootstrap using a GitHub org, Gitlab and other git providers, see [Bootstraping](https://fluxcd.io/flux/installation/bootstrap/).

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=home-cluster \
  --branch=main \
  --path=./clusters/home-cluster \
  --personal
```

The output is similar to:

```sh
► connecting to github.com
✔ repository created
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
✔ install completed
► configuring deploy key
✔ deploy key configured
► generating sync manifests
✔ sync manifests pushed
► applying sync manifests
◎ waiting for cluster sync
✔ bootstrap finished
```
The bootstrap command above does the following:

- Creates a git repository *home-cluster* on your GitHub account.
- Adds Flux component manifests to the repository.
- Deploys Flux Components to your Kubernetes Cluster. 
- Configures Flux components to track the path /clusters/home-cluster/ in the repository

By default **Flux** installs 4 components: **source-controller**, **kustomize-controller**, **helm-controller** and **notification-controller**, you can check that all of them are properly running by looking in the *flux-system* namespace.

```sh
❯ k get all -n flux-system
NAME                                          READY   STATUS    RESTARTS   AGE
pod/helm-controller-5f964c6579-z44r9          1/1     Running   0          6d18h
pod/kustomize-controller-9c588946c-6h9fd      1/1     Running   0          6d18h
pod/notification-controller-76dc5d768-z47jw   1/1     Running   0          6d18h
pod/source-controller-6c49485888-gl6dz        1/1     Running   0          6d18h

NAME                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/notification-controller   ClusterIP   10.97.135.57    <none>        80/TCP    6d18h
service/source-controller         ClusterIP   10.97.126.173   <none>        80/TCP    6d18h
service/webhook-receiver          ClusterIP   10.107.72.214   <none>        80/TCP    6d18h

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helm-controller           1/1     1            1           6d18h
deployment.apps/kustomize-controller      1/1     1            1           6d18h
deployment.apps/notification-controller   1/1     1            1           6d18h
deployment.apps/source-controller         1/1     1            1           6d18h

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/helm-controller-5f964c6579          1         1         1       6d18h
replicaset.apps/kustomize-controller-9c588946c      1         1         1       6d18h
replicaset.apps/notification-controller-76dc5d768   1         1         1       6d18h
replicaset.apps/source-controller-6c49485888        1         1         1       6d18h
```

And now, to move to the next steps, we can just clone the **GIT** repository we just created to start deploying apps and configuration onto our Cluster.

```sh
git clone git@github.com:cc250080/home-cluster.git
cd home-cluster
```

### Flux Structure

To structure the repository we use a mono-repo approach, the flux documentation [Ways of structuring your repositories](https://fluxcd.io/flux/guides/repository-structure/) is very helpful here specially the following example which I took as a [baseline](https://github.com/fluxcd/flux2-kustomize-helm-example).
