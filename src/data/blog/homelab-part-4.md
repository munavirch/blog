---
author: Munavir Chavody
pubDatetime: 2025-10-08T15:22:00Z
modDatetime: 2025-10-08T16:52:45.934Z
title: "Journey to Bare-Metal Kubernetes from Scratch – Part 4: Cluster Up and Running"
slug: journey-to-bare-metal-part-4-kubernetes-cluster
featured: true
draft: false
tags:
  - journey-to-bare-metal
  - bare-metal
  - kubernetes
  - homelab
  - proxmox
description: "Fourth part of my 'bare-metal Kubernetes journey': cluster up and running."
---

## Table of contents

## Introduction

> Recap: 
    <br>- [Part 1](/blog/posts/journey-to-bare-metal-part-1-building-the-rig), I discussed my experience sourcing and assembling the hardware. 
    <br>- [Part 2](/blog/posts/journey-to-bare-metal-part-2-virtualisation-layer), we went through how to set up Proxmox as virtualisation layer.
    <br>- [Part 3](/blog/posts/journey-to-bare-metal-part-3-kubernetes-nodes), we created a base image for Kubernetes nodes.

Alright, time to bring our cluster to life 😎

We now have a solid foundation to run our Kubernetes (K8s) nodes. The plan is to host the K8s cluster on Proxmox VMs, using our prepared template to quickly spin up the instances.

There will be a total of three VMs: 
- `master-01` will host control plane
- `worker-01` and `worker-02` as worker nodes. 

## Launching VMs

Let's quickly clone 3 VMs from the template.

```bash
# Find out the ID of the template, in my case it's 1002
qm list

# Launch master-01
qm clone 1002 7001 --name master-01 --full

# Now specify options for cloud-init. Be cautious with the password!
qm set 7001 --ciuser ubuntu --cipassword 'ubuntu' --ipconfig0 ip=dhcp

# Optionally, set up SSH keys for headless login.
qm set 7001 --sshkey ~/.ssh/id_rsa.pub

# Start the VM
qm start 7001
```

See, there's a `--full` flag in clone command. It specifies that Proxmox should clone the full disk and treat it as a standalone disk. The alternative to this is a linked clone, meaning Proxmox will track the diff to the source disk resulting in a lower space usage. Full cloned VMs are relatively faster since the disk is independent of the source disk, but takes more storage. So, to balance out, we'll use full clone for master nodes and linked clone for worker nodes.

Now get the IP either from Proxmox UI or via `qm guest` command. Here's what I have in my server.

```bash
root@pve:/etc/pve/qemu-server# qm guest cmd 7001 network-get-interfaces | jq '.[]."ip-addresses"[] | select(."ip-address-type"=="ipv4") | ."ip-address"'
# local
"127.0.0.1"
# LAN IP
"192.168.0.7"
```

Now let's repeat the above steps to launch 2 worker nodes with hostnames `worker-01` and `worker-02` - keep in mind to remove the `--full` flag.

## Bootstrapping the Control Plane

> Kubeadm is a tool built to provide kubeadm init and kubeadm join as best-practice "fast paths" for creating Kubernetes clusters. kubeadm performs the actions necessary to get a minimum viable cluster up and running. By design, it cares only about bootstrapping, not about provisioning machines.

Kubeadm provide init command to bootstrap a machine to a Kubernetes node. Let's do that and go through the logs in detail.

```bash
sudo kubeadm init --apiserver-advertise-address 192.168.0.7 --pod-network-cidr 10.244.0.0/16
```

Outputs:

```bash
[init] Using Kubernetes version: v1.34.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W1007 14:46:23.310795    1681 checks.go:830] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended to use "registry.k8s.io/pause:3.10.1" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master-01] and IPs [10.96.0.1 192.168.0.7]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master-01] and IPs [192.168.0.7 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master-01] and IPs [192.168.0.7 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 501.437027ms
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.0.7:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 5.606949978s
[control-plane-check] kube-scheduler is healthy after 6.457968013s
[control-plane-check] kube-apiserver is healthy after 8.001042238s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master-01 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master-01 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 8vzuea.jo8g3ws3mf006jsb
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.7:6443 --token <token> \
	--discovery-token-ca-cert-hash sha256:<hash>
```

- `[preflight]` refers to the checks kubeadm will perform before starting the installation to make sure everything is in order.
- That warning right after preflight step just means that we are using older version of pause container, nothing fatal.
- `[certs]` stage generates the certificates required for various components.
- `[kubeconfig]` writes various user credentials config including default admin, super admin and control plan users.
- Upcoming stages configure control plane components and start the same using [static pods ↗](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/).
- Final stage sets up bootstrap which can be used to add nodes to the cluster.
- Also, note down how we can use the admin.conf as our kubeconfig file and the command to join the nodes to the cluster.

Now, I'll setup kubeconfig file as per the instruction and see the running pods.

```bash
ubuntu@master-01:/etc/kubernetes$ kubectl get pods -A
NAMESPACE     NAME                                READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bc5c9577-d8xv8            0/1     Pending   0          20m
kube-system   coredns-66bc5c9577-wvjdp            0/1     Pending   0          20m
kube-system   etcd-master-01                      1/1     Running   0          20m
kube-system   kube-apiserver-master-01            1/1     Running   0          20m
kube-system   kube-controller-manager-master-01   1/1     Running   0          20m
kube-system   kube-proxy-bnvjk                    1/1     Running   0          20m
kube-system   kube-scheduler-master-01            1/1     Running   0          20m
```

Voila! Our control plane is up and running! Wait, there's something wrong here. Our CoreDNS pods are not healthy it's pending. Looking at the pod events I see this message:

```
0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

It says there's a `NotReady` taint for the node. This is because we have not installed a CNI plugin to feciliate connectivity using pod IPs. By default nodes will have `NotReady` taint and once the daemonset for the node comes up, it removes the taint. Let's install the same and see what happens.

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

But, we have not specified any CIDR info to flannel, right? Well, the pod network CIDR we mentioned while kubeadm init is the default CIDR that flannel use. Now let's see if CoreDNS pods are scheduled!

```bash
ubuntu@master-01:~$ kubectl get pods -A
NAMESPACE      NAME                                READY   STATUS    RESTARTS      AGE
kube-flannel   kube-flannel-ds-7947k               1/1     Running   0             27s
kube-system    coredns-66bc5c9577-d8xv8            1/1     Running   0             20h
kube-system    coredns-66bc5c9577-wvjdp            1/1     Running   0             20h
kube-system    etcd-master-01                      1/1     Running   1 (11m ago)   20h
kube-system    kube-apiserver-master-01            1/1     Running   1 (11m ago)   20h
kube-system    kube-controller-manager-master-01   1/1     Running   1 (11m ago)   20h
kube-system    kube-proxy-bnvjk                    1/1     Running   1 (11m ago)   20h
kube-system    kube-scheduler-master-01            1/1     Running   1 (11m ago)   20h
```

Again, Voila! It's up and running.

## Launching Worker Nodes

Let's configure the worker nodes that we launched earlier. Login to each node and run the `kubeadm join` command we got from running the `kubeadm init` command.

```bash
sudo kubeadm join 192.168.0.7:6443 --token <token>  --discovery-token-ca-cert-hash <hash>
```

Now, let's see if the node is available in the cluster.

```bash
ubuntu@master-01:~$ kubectl get nodes
NAME        STATUS   ROLES           AGE     VERSION
master-01   Ready    control-plane   20h     v1.34.1
worker-01   Ready    <none>          2m58s   v1.34.1
```

Yes! We did it. One small thing before we wind up - since we have joined our node to the cluster, Kubernetes will recognizes which node is `worker-01`. However, if the cluster or VM is restarted, Kubernetes won’t be able to map worker-01 to the specific VM (7002) automatically.

To fix this we need to add `spec.providerID` which uniquely identify the Proxmox VM which is respeonsible for `worker-01`. Let's do `kubectl edit worker-01` and add `providerID` under `.spec` and update the value to `proxmox://7002`.

Now repeat the same steps for our `worker-02` to join the cluster.

## Outro

Having the Proxmox VM template made the cluster bootstrap process very easy because the nodes came with all the required tooling pre-installed. And setting up the cluster is fairly straightforward with the help of `kubeadm`.

**Up Next**: We’ll automate storage setup for our cluster.