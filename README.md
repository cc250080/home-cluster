# flux-cluster
Welcome to my **really** highly opinionated Repo for my GitOps home-lab. In this repo I install a basic Vanilla [Kubernetes](https://kubernetes.io/docs/home/) and I use [Flux](https://fluxcd.io/flux/) to manage its state.

## 👋 Introduction
My personal Goal with this project is to have an easy and elegant way to manage applications that I want to run in my kubernetes home-lab while at the same time use it to keep learning the intrinsicacies of Kubernetes and GitOps. That is why I took some choices like for example installing Vanilla Kubernetes by hand. You won't find here *Ansible* playbooks or other automatisms to install Kubernetes, Flux and its tools. For now I choose to install everything by hand and learn and interiorize during the process. It is my goal to take out any abstraction on top of the basic Kubernetes components while at the same time enjoying a useful GitOps installation.

## ✨ Features

- Documented **manual** Installation of Kubernetes v1.28 mono-node cluster in Debian 12 *BookWorm*
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
- [ ] have a DNS server that supports split DNS (e.g. Pi-Hole) deployed somewhere outside your cluster **ON** your home network.

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
    sudo vi /etc/containerd/config.toml
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

### 🔧 Install Kubernetes with KubeAdm

When we already have a CRI installed(containerd) and starting properly using systemd it is the time to install K8s, for that I wont be documenting the steps, just follow the instructions from the official Kubernetes [documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

Then, after installing kubeadm, kubectl and kubelet and perform the **kubeadm init** we can ensure everything is *almost* up and ready in our cluster:

```sh
~ on  main [!?] 
❯ k get nodes
NAME       STATUS   ROLES           AGE   VERSION
bacterio   UnReady    control-plane   5d    v1.28.3
```
In order to get a Ready state, we still have to install our Pod Network Add-on or **CNI**.

### Installing the Container Network Interface (CNI) add-on

In order to install **Cilium** it is enough to follow the [Cillium Quick Installation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/), we will ned with the *Cilium CLI* tool and *Cilium* installed in our Cluster.

And after that the Cluster will become Ready.

```sh
~ on  main [!?] 
❯ k get nodes
NAME       STATUS   ROLES           AGE   VERSION
bacterio   Ready    control-plane   5d    v1.28.3
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


