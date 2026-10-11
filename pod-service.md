# Kubernetes Hands-on Guide: Installation, kubeadm, CRI-O, Pods & Services

Objective: Install Kubernetes tools on Linux, configure a multi-node cluster using kubeadm and CRI-O, deploy your first Pod, expose it using Kubernetes Services, and troubleshoot common issues.

## 1. Kubernetes tools overview

| Tool           | Purpose                                           | Where installed                |
| -------------- | ------------------------------------------------- | ------------------------------ |
| kubeadm        | Bootstraps and manages cluster setup and upgrades | Control plane and worker nodes |
| kubelet        | Runs on nodes and manages Pods                    | All nodes                      |
| kubectl        | CLI to communicate with Kubernetes API            | Administrator machine          |
| CRI-O          | Container runtime that runs containers            | All nodes                      |
| Minikube       | Creates local Kubernetes clusters                 | Laptop or development VM       |
| kube-apiserver | Entry point for Kubernetes API operations         | Control plane                  |
| etcd           | Stores Kubernetes cluster state                   | Control plane                  |
| kube-proxy     | Implements Service networking where used          | Typically all nodes            |

## 2. Kubernetes architecture

## 3. Minikube vs kubeadm

| Feature    | Minikube                          | kubeadm                           |
| ---------- | --------------------------------- | --------------------------------- |
| Main use   | Learning and development          | Multi-node cluster setup          |
| Setup      | Simple                            | Manual configuration              |
| Nodes      | Usually single, supports multiple | Single or multiple                |
| Runtime    | Configurable                      | CRI-compatible runtime            |
| Control    | Mostly automated                  | More infrastructure control       |
| Production | Not recommended                   | Can bootstrap production clusters |
| Example    | Local laptop lab                  | Linux VMs or bare-metal servers   |

## 4. Install Kubernetes using kubeadm and CRI-O

Lab infrastructure

| Server        | Hostname    | Example private IP |
| ------------- | ----------- | ------------------ |
| Control plane | k8s-control | 192.168.1.10       |
| Worker 1      | k8s-worker1 | 192.168.1.11       |
| Worker 2      | k8s-worker2 | 192.168.1.12       |

Use Ubuntu 24.04 LTS on all three servers. The commands below illustrate a Kubernetes v1.35 installation; align the minor version with your chosen supported release.

### Step 1 — Prepare all nodes

```
sudo apt update
sudo apt install -y curl ca-certificates \
  gpg apt-transport-https

# Disable swap for the lab
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Networking settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Set a unique hostname on each server using `sudo hostnamectl set-hostname k8s-control`, replacing the name on each worker.

### Step 2 — Install CRI-O

Run on all nodes:

```
export K8S_VERSION=v1.35

curl -fsSL \
https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$K8S_VERSION/deb/Release.key \
| sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
```

Create the keyring directory first if needed:

```
sudo mkdir -p /etc/apt/keyrings
```

Then configure the repository:

```
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] \
https://download.opensuse.org/repositories/isv:/cri-o:/stable:/$K8S_VERSION/deb/ /" \
| sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt update
sudo apt install -y cri-o

sudo systemctl enable --now crio
sudo systemctl status crio
```

Check repository availability and release compatibility before running these commands on production systems.

### Step 3 — Install kubeadm, kubelet and kubectl

Run on all nodes:

```
export K8S_MINOR=v1.35

curl -fsSL \
https://pkgs.k8s.io/core:/stable:/$K8S_MINOR/deb/Release.key \
| sudo gpg --dearmor \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/$K8S_MINOR/deb/ /" \
| sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

The kubelet may restart until the cluster is initialized. That is expected.

### Step 4 — Initialize the control plane

Run only on `k8s-control`:

```
sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.10 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///var/run/crio/crio.sock
```

The Pod CIDR is an example for a compatible CNI configuration. Ensure it does not overlap the node or VPC network.

Configure kubectl:

```
mkdir -p $HOME/.kube

sudo cp /etc/kubernetes/admin.conf \
  $HOME/.kube/config

sudo chown $(id -u):$(id -g) \
  $HOME/.kube/config

kubectl get nodes
```

Install a compatible CNI network plugin such as Calico using its current official installation instructions. Match its Pod IP pool to your chosen CIDR.

### Step 5 — Join worker nodes

Generate a join command on the control plane:

```
kubeadm token create --print-join-command
```

Execute the generated `kubeadm join ...` command on both workers with `sudo`, adding the CRI-O socket option if necessary.

Verify:

```
kubectl get nodes -o wide
kubectl get pods -A
```

Expected after CNI installation:

```
NAME          STATUS   ROLES           AGE
k8s-control   Ready    control-plane   10m
k8s-worker1   Ready    <none>          5m
k8s-worker2   Ready    <none>          5m
```

For production, additionally plan for high availability, firewall rules, secure API access, RBAC, etcd backups, node maintenance and monitoring.

## 5. Create your first Kubernetes Pod

Create `pod.yaml`:

```

apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80

```

Deploy:

```
kubectl apply -f pod.yaml
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl logs nginx-pod
```

Test the application from your local machine through port forwarding:

```
kubectl port-forward pod/nginx-pod 8080:80
```

Open `http://localhost:8080`.

## 6. Pod lifecycle

| Pod phase/status | Meaning                                                                 |
| ---------------- | ----------------------------------------------------------------------- |
| Pending          | Waiting for scheduling or container setup                               |
| Running          | Bound to a node; at least one container running, starting or restarting |
| Succeeded        | All containers completed successfully                                   |
| Failed           | Containers terminated and at least one failed                           |
| Unknown          | Pod state cannot be determined                                          |
| CrashLoopBackOff | Container restart backoff; not a Pod phase                              |
| ImagePullBackOff | Image cannot currently be pulled; not a Pod phase                       |

Troubleshooting commands:

```
kubectl describe pod nginx-pod
kubectl logs nginx-pod --previous
kubectl get events --sort-by=.lastTimestamp
kubectl get nodes
kubectl get pods -A
```

## 7. Kubernetes Services and types

A Service provides a stable network identity to access a set of Pods.

| Service type | Accessibility                         | Example          |
| ------------ | ------------------------------------- | ---------------- |
| ClusterIP    | Within cluster                        | Internal backend |
| NodePort     | Node IP and allocated port            | Lab/testing      |
| LoadBalancer | External load balancer when supported | Cloud web app    |

### ClusterIP example

Create `service.yaml`:

```

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP

```

Apply and verify:

```
kubectl apply -f service.yaml

kubectl get svc
kubectl describe svc nginx-service
kubectl get endpoints nginx-service
kubectl get endpointslices \
  -l kubernetes.io/service-name=nginx-service
```

### NodePort example

Change the Service type and specify a node port:

```
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Access through `http://NODE_IP:30080`, provided the firewall permits it.

### LoadBalancer example

```
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

On AWS EKS or Azure AKS, an appropriate cloud integration can provision a load balancer. On bare-metal kubeadm clusters, an additional implementation such as MetalLB is normally required.

## 8. Endpoints and Service Discovery

Endpoints represent the backend addresses associated with a Service. Modern Kubernetes uses EndpointSlices to manage backend endpoint information.

Service discovery is typically provided through CoreDNS.

Example: A Pod in the `default` namespace can access the Service using:

```
nginx-service.default.svc.cluster.local
```

Test DNS and connectivity:

```
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600

kubectl exec dns-test -- \
  nslookup nginx-service

kubectl exec dns-test -- \
  wget -qO- http://nginx-service
```

## 9. Important troubleshooting commands

| Command                         | Purpose                 |
| ------------------------------- | ----------------------- |
| `kubectl get nodes`             | Check node readiness    |
| `kubectl get pods -A`           | List all Pods           |
| `kubectl describe pod POD`      | Inspect Pod issues      |
| `kubectl logs POD`              | Read container logs     |
| `kubectl get svc`               | List Services           |
| `kubectl get endpointslices`    | Inspect EndpointSlices  |
| `kubectl get events -A`         | View cluster events     |
| `sudo systemctl status kubelet` | Check kubelet           |
| `sudo systemctl status crio`    | Check CRI-O             |
| `sudo journalctl -u kubelet -f` | Follow kubelet logs     |
| `sudo crictl ps -a`             | List runtime containers |

## 10. Practical lab checklist

Lab progress

0 of 10 completed

Prepare three Ubuntu Linux servers

Install kubeadm, kubelet, kubectl and CRI-O

Initialize the control plane with kubeadm

Install the CNI plugin and join worker nodes

Create and deploy nginx-pod

Inspect Pod lifecycle, logs and events

Create a ClusterIP Service

Convert the Service to NodePort and test access

Inspect EndpointSlices and verify DNS discovery

Clean up lab resources

Reset checklist

Recommended learning sequence: Linux networking → container runtimes → kubeadm cluster installation → Pods → Labels and Selectors → Services → EndpointSlices → CoreDNS → Deployments.

For the next practical session, use a Deployment with three Nginx replicas instead of three manually created Pods. Then expose them through ClusterIP and NodePort Services to demonstrate load distribution, Pod replacement, and self-healing.
