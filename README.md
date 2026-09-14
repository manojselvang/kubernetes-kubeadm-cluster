# Building Kubernetes Cluster from scratch using Kubeadm

A hands-on guide for building a small Kubernetes cluster from scratch using `kubeadm`.

> **Lab setup:** 1 control-plane node + 2 worker nodes  
> **OS:** Ubuntu 22.04  
> **Container runtime:** containerd  
> **CNI:** Calico

This project is meant to help you understand what actually happens when a Kubernetes cluster is built, instead of treating Kubernetes as a black box.

## What You'll Learn

- How the control plane and worker nodes fit together
- What `kubeadm`, `kubelet`, `kubectl`, and `containerd` do
- Why swap is disabled
- How pod networking works with a CNI
- How worker nodes join a cluster
- How Services, Ingress, and kube-proxy fit into traffic flow
- How to verify and troubleshoot a basic cluster

## Architecture

```text
                         Kubernetes Cluster
                                |
                 +--------------+--------------+
                 |                             |
          Control Plane                    Worker Nodes
       k8s-control-plane              +--------+--------+
                                      |                 |
                                k8s-worker-1      k8s-worker-2
```

## Prerequisites

- 3 Linux machines
- Ubuntu 22.04 recommended
- At least 2 CPUs and 2 GB RAM per machine
- SSH access
- Network connectivity between all nodes

AWS EC2 can be used for the lab, but the Kubernetes installation itself is not AWS-specific.

## Contents

- [What We Are Building](#1-what-we-are-building)
- [Components](#2-know-these-components-first)
- [Prepare the Machines](#3-prepare-the-machines)
- [AWS EC2 Setup](#4-if-you-are-using-aws-ec2)
- [Prepare Every Node](#5-prepare-every-node)
- [Install Containerd](#6-install-containerd)
- [Install Kubernetes Tools](#7-install-kubernetes-tools)
- [Initialize the Control Plane](#8-initialize-the-control-plane)
- [Configure kubectl](#9-configure-kubectl-for-your-user)
- [Install Calico](#10-install-calico)
- [Join Worker Nodes](#11-add-the-worker-nodes)
- [Recover the Join Command](#12-what-if-you-lost-the-join-command)
- [Verify the Cluster](#13-verify-the-cluster)
- [Test Deployment](#14-run-a-small-test-application)
- [Kubernetes Traffic Flow](#15-how-network-traffic-reaches-a-pod)
- [Two-Node Lab](#16-using-only-two-machines)
- [Troubleshooting](#17-common-problems)
- [Cleanup](#18-cleaning-up)
- [Check Your Understanding](#19-questions-you-should-be-able-to-answer)

---

## 1. What We Are Building

A Kubernetes cluster has two main types of machines:

- **Control plane** — keeps the desired state of the cluster and makes scheduling and management decisions.
- **Worker nodes** — run the application workloads.

For this exercise:

```text
                 Kubernetes Cluster
                       |
             +---------+---------+
             |                   |
       Control Plane          Workers
       k8s-control-plane      worker-1
                              worker-2
```

We will use:

- Ubuntu 22.04
- `containerd` as the container runtime
- `kubeadm` to bootstrap the cluster
- `kubectl` to operate it
- Calico as the pod network

The process is:

1. Prepare three Linux machines.
2. Install and configure the container runtime.
3. Install the Kubernetes packages.
4. Initialize the control plane.
5. Install the pod network.
6. Join the workers.
7. Check the cluster.
8. Deploy a small test workload.

---

### 2. Know These Components First

### Containerd

Kubernetes does not directly start containers. It relies on a container runtime for that job.

Here we use **containerd**. When Kubernetes needs a container created, started, stopped, or removed, the kubelet works with the runtime to make that happen.

### Kubelet

`kubelet` runs on every node.

Its job is to make sure the workloads assigned to that node are actually running. It communicates with the Kubernetes control plane and the container runtime.

### Kubeadm

`kubeadm` is used to bootstrap the cluster.

You normally use it for operations such as:

- initializing a control plane with `kubeadm init`
- adding nodes with `kubeadm join`
- generating a new worker join command when necessary

It is not the command you will use for normal day-to-day cluster administration.

### Kubectl

`kubectl` is the main command-line interface for interacting with Kubernetes.

After the cluster is running, commands such as these become your normal tools:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployments
```

### CNI / Pod Network

Pods need networking. A pod on one node must be able to communicate with a pod on another node.

A CNI (Container Network Interface) plugin provides that networking layer.

For this lab, we use **Calico**.

Without a working CNI, a freshly initialized kubeadm cluster will not behave like a usable multi-node cluster.

---

## 3. Prepare the Machines

You need three Linux machines for the full exercise:

```text
k8s-control-plane
k8s-worker-1
k8s-worker-2
```

A practical starting point is at least:

- 2 CPUs per machine
- 2 GB RAM per machine
- SSH access
- network connectivity between all three machines

For a learning environment, putting the machines in the same AWS VPC and subnet makes networking easier to understand.

---

## 4. If You Are Using AWS EC2

If you already have Linux machines, skip this section.

Create three EC2 instances using Ubuntu 22.04 LTS.

For a lab, a `t3.medium` is a reasonable starting point because Kubernetes components need some memory and the extra headroom makes troubleshooting less frustrating.

### Network setup

Make sure the instances can communicate with one another.

At minimum, you need:

- SSH access on port 22 from your own IP
- Internal communication between the three cluster machines

For a temporary learning environment, allowing traffic from the same security group is a simple approach.

Importanat Note: All nodes must attached to same security group for this lab. And traffic must be allowed for the nodes from this same security group like below example

| Type | Protocol | Port | Source |
|---|---|---:|---|
| SSH | TCP | 22 | My IP |
| All traffic | All | All | `sg-0123456789abcdef` (Security Group) |

Do not blindly copy this model into production. In a real environment, restrict traffic to the ports and sources actually required by your cluster.

After the instances are running, record their private IP addresses.

For example:

```text
Control plane: 10.0.1.15
Worker 1:      10.0.1.16
Worker 2:      10.0.1.17
```

These are examples only. Use the addresses assigned to your machines.

---

## 5. Prepare Every Node

The following steps must be completed on **all three machines**.

SSH into each machine and work through the section.

## 5.1 Update the operating system

```bash
sudo apt update
sudo apt upgrade -y
```

This gets the base operating system up to date before installing the Kubernetes components.

---

## 5.2 Give each machine a useful hostname

A unique hostname is not strictly required for the cluster to work, but it makes administration much easier.

### Control plane

```bash
sudo hostnamectl set-hostname k8s-control-plane
```

### Worker 1

```bash
sudo hostnamectl set-hostname k8s-worker-1
```

### Worker 2

```bash
sudo hostnamectl set-hostname k8s-worker-2
```

Reconnect if necessary and verify:

```bash
hostname
```

---

## 5.3 Turn off swap

Kubernetes expects swap to be disabled for this setup.

Disable it immediately:

```bash
sudo swapoff -a
```

Then prevent the swap entry in `/etc/fstab` from enabling it again after a reboot:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

You can check the current state with:

```bash
free -h
```

---

## 5.4 Load the kernel modules

Kubernetes networking requires Linux kernel features that are not necessarily loaded by default.

Create the module configuration:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Load them now:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 5.5 Configure IP forwarding and bridge networking

Create the required sysctl configuration:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply it:

```bash
sudo sysctl --system
```

The important idea here is that Kubernetes needs the host to forward and correctly process traffic moving between interfaces, bridges, containers, and pods.

---

## 6. Install Containerd

Do this on every node.

```bash
sudo apt install -y containerd
```

Create the configuration directory:

```bash
sudo mkdir -p /etc/containerd
```

Generate a default configuration:

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Open it:

```bash
sudo nano /etc/containerd/config.toml
```

Find:

```text
SystemdCgroup = false
```

Change it to:

```text
SystemdCgroup = true
```

The container runtime and kubelet should use compatible cgroup management. Using systemd here avoids a common source of kubelet/runtime configuration problems.

Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Check it:

```bash
sudo systemctl status containerd
```

You want the service to be active.

---

## 7. Install Kubernetes Tools

Again, run these commands on **all three nodes**.

Install the packages needed to add the Kubernetes repository:

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Create the keyring directory if required:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Add the Kubernetes repository key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update package information:

```bash
sudo apt update
```

Install:

```bash
sudo apt install -y kubelet kubeadm kubectl
```

Prevent an ordinary package upgrade from changing these versions unexpectedly:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Check the installed tools:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

> **Version note:** This example uses the Kubernetes `v1.30` repository because that is the version used by this lab. Kubernetes releases move forward regularly. If you build this lab with another release, use a repository and CNI version that are compatible with that release instead of blindly mixing versions.

---

## 8. Initialize the Control Plane

From this point, the commands are no longer identical on every machine.

The next steps happen **only on the control-plane node**.

Run:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

The command bootstraps the Kubernetes control plane.

The `--pod-network-cidr` option reserves an address range for pod networking. The range shown above is being used because it matches the networking configuration used in this example.

At the end of the command, kubeadm prints a `kubeadm join` command.

It will look roughly like this:

```bash
sudo kubeadm join 10.0.1.15:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**Do not copy the example values above.**

Copy the actual command printed by your own `kubeadm init` output. You will use it on the worker nodes.

---

## 9. Configure Kubectl for Your User

`kubeadm init` creates an administrator kubeconfig at:

```text
/etc/kubernetes/admin.conf
```

Copy it into your user's kubeconfig location:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Now test access:

```bash
kubectl get nodes
```

At this point, seeing the control plane as `NotReady` is normal if the pod network has not been installed yet.

Do not start randomly changing settings just because the node says `NotReady`. We have not completed the networking setup yet.

---

## 10. Install Calico

Run this on the control plane.

For the version used by this example:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Give the components some time to start.

Check the nodes:

```bash
kubectl get nodes
```

Also inspect the system pods:

```bash
kubectl get pods -n kube-system
```

You should eventually see the relevant system pods running.

If the control plane changes to:

```text
Ready
```

the basic cluster networking is working.

---

## 11. Add the Worker Nodes

Now move to `k8s-worker-1`.

Run the exact `kubeadm join` command produced when you initialized the control plane.

For example:

```bash
sudo kubeadm join 10.0.1.15:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```
Ensure to run the above command with `sudo`. Else it will throw below error. 

`[ERROR IsPrivilegedUser]: user is not running as root`.

Then repeat the same process on `k8s-worker-2`.

Each worker contacts the Kubernetes API server on the control plane and completes the bootstrap process.

Go back to the control plane and run:

```bash
kubectl get nodes
```

You should eventually have three nodes:

```text
NAME                 STATUS   ROLES
k8s-control-plane    Ready    control-plane
k8s-worker-1         Ready    <none>
k8s-worker-2         Ready    <none>
```

The exact output can vary slightly depending on the Kubernetes version.

---

## 12. What If You Lost the Join Command?

You do not have to rebuild the cluster.

On the control plane, run:

```bash
sudo kubeadm token create --print-join-command
```

That generates a fresh join command.

Use the new command on the worker node.

If a previous token has expired, generating a new one is the normal way to continue.

---

## 13. Verify the Cluster

From the control plane:

```bash
kubectl get nodes
```

All three nodes should eventually report `Ready`.

Then check all namespaces:

```bash
kubectl get pods -A
```

Look for pods that are stuck in states such as:

```text
Pending
CrashLoopBackOff
ImagePullBackOff
```

A newly created cluster may take a little time to settle, so do not panic over a transient state. If a pod remains unhealthy, then investigate it.

Useful commands include:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

and:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

## 14. Run a Small Test Application

Now let's make sure the cluster can actually run an application.

Create a two-replica nginx deployment:

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
```

Check the pods:

```bash
kubectl get pods -o wide
```

The `-o wide` output includes the node on which each pod is running.

This gives you a simple demonstration of Kubernetes scheduling workloads onto nodes.

When finished, remove the test:

```bash
kubectl delete deployment nginx-test
```

---

## 15. How Network Traffic Reaches a Pod

This part is worth understanding because Kubernetes networking can look confusing when you first encounter it.

A user does not normally connect directly to a pod IP.

Traffic can enter the cluster through mechanisms such as:

- a cloud load balancer
- a NodePort
- an Ingress controller

An Ingress is not the same thing as kube-proxy.

For example, an Ingress controller can inspect an HTTP request and decide which Kubernetes Service should receive it.

The Service then provides a stable endpoint in front of a group of pods.

`kube-proxy` is involved in implementing Service traffic routing on nodes. Depending on the configuration, this can involve networking rules such as iptables or IPVS.

A simplified flow looks like:

```text
Client
  |
  v
Load Balancer / NodePort / Ingress
  |
  v
Kubernetes Service
  |
  v
Service routing rules
  |
  v
Pod
```

The important distinction is:

- **Ingress** handles higher-level routing, commonly HTTP/HTTPS.
- **Service** provides a stable way to reach a set of pods.
- **kube-proxy/networking rules** help implement Service traffic routing.
- **The CNI** provides pod networking between nodes.

These components solve different problems.

---

## 16. Using Only Two Machines

The full example uses three machines because it gives you one control plane and two workers.

If you only have two Linux machines, you can still build a small lab:

```text
Machine 1 -> Control Plane
Machine 2 -> Worker
```

In that case, simply skip everything referring to `k8s-worker-2`.

The Kubernetes installation steps remain the same.

The main requirement is that both machines can communicate reliably over the network.

If you are using your own physical machines or VMs, make sure local firewalls are not blocking the traffic required by Kubernetes.

---

## 17. Common Problems

## Node remains NotReady

Start with:

```bash
kubectl get pods -n kube-system
```

Look for Calico or other system pods that are not healthy.

Also check:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

A common cause in a fresh setup is a networking problem.

---

## `kubeadm join` times out

The worker needs to reach the API server on the control plane.

Check:

- the control-plane IP address
- AWS Security Group rules
- local firewall rules
- whether the API server is actually running
- whether the worker can reach the control plane over the required port

On the control plane:

```bash
kubectl get pods -n kube-system
```

---

## Kubelet is not healthy

Check:

```bash
sudo systemctl status kubelet
```

Then inspect recent logs:

```bash
sudo journalctl -u kubelet -xe
```

Also verify that containerd is running:

```bash
sudo systemctl status containerd
```

If you configured containerd manually, confirm that:

```text
SystemdCgroup = true
```

and restart containerd after making the change.

---

## Worker join token no longer works

Generate another one:

```bash
sudo kubeadm token create --print-join-command
```

Then run the new command on the worker.

---

Yes — you have **three** practical problems documented. If you want to paste them **at the very bottom of your existing `## 17. Common Problems` section**, use this exactly:

````markdown
---

### `kubeadm join` fails with `user is not running as root`

If you run the `kubeadm join` command without `sudo`, you may see:

```text
[ERROR IsPrivilegedUser]: user is not running as root
````

Run the join command with `sudo`:

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

`kubeadm join` requires root privileges because it needs to configure the worker node and kubelet.

---

### `kubeadm join` gets stuck at `Running pre-flight checks`

Sometimes the join command may appear to hang at:

```text
[preflight] Running pre-flight checks
```

Check the AWS Security Group rules and make sure the worker can communicate with the control plane.

For this lab, all three nodes use the same Security Group:

| Type        | Protocol | Port | Source                                 |
| ----------- | -------- | ---: | -------------------------------------- |
| SSH         | TCP      |   22 | My IP                                  |
| All traffic | All      |  All | `sg-0123456789abcdef` (Security Group) |

---

## 18. Cleaning Up

If you are finished with the lab, you can reset the Kubernetes installation.

On a worker:

```bash
sudo kubeadm reset -f
```

On the control plane:

```bash
sudo kubeadm reset -f
sudo rm -rf $HOME/.kube
```

If the machines are AWS EC2 instances and you no longer need them, terminate them so you do not continue paying for unused resources.

---

## 19. Questions You Should Be Able to Answer

Do not move on just because the commands worked. If you cannot explain these without looking at the notes, you probably memorized the procedure rather than learning it.

1. What is the control plane responsible for?
2. What actually runs the application containers on a worker node?
3. Why does Kubernetes need a container runtime?
4. What is the difference between `kubeadm` and `kubectl`?
5. Why is swap disabled for this setup?
6. What does the CNI provide?
7. Why did the control-plane node initially show `NotReady`?
8. What does `kubeadm init` do?
9. Where does the worker's `kubeadm join` command come from?
10. What happens if the original join token is no longer usable?
11. What is the difference between an Ingress, a Service, and kube-proxy?
12. What does `kubectl get pods -A` tell you?
13. Why is `SystemdCgroup = true` configured in containerd?
14. How would you start troubleshooting a worker that cannot join the cluster?

If you can explain those in your own words, you understand the basic kubeadm workflow rather than just following a recipe.

---

# References

Use the official documentation when adapting this lab to a newer Kubernetes release.

- Kubernetes kubeadm installation documentation
- Kubernetes kubeadm cluster creation documentation
- Kubernetes release information
- Calico Kubernetes installation documentation
- containerd documentation
- AWS EC2 documentation

The original source material for this rewrite is the uploaded kubeadm runbook. fileciteturn0file0

---

## References

- [Kubernetes kubeadm installation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Creating a Kubernetes cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes releases](https://kubernetes.io/releases/)
- [Calico Kubernetes quickstart](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)
- [containerd documentation](https://containerd.io/docs/)
- [AWS EC2 documentation](https://docs.aws.amazon.com/ec2/)

## Notes

This README is intended as a **learning/lab guide**, not a production Kubernetes installation guide.

Before using this setup in a real environment, review Kubernetes version compatibility, CNI compatibility, firewall rules, HA requirements, node sizing, storage, and cluster security.
