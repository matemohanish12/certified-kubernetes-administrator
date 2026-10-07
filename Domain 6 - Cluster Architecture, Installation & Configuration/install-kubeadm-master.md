#### Step 1: Setup containerd
This step prepares the node for Kubernetes by enabling the kernel modules and network settings that container runtimes need. `overlay` and `br_netfilter` are required for container filesystem overlays and bridge traffic handling. The `sysctl` values enable IPv4/IPv6 forwarding and bridge-based iptables filtering so pods can communicate correctly. After that, we install `containerd`, generate its default configuration, and enable systemd cgroup support so Kubernetes can manage container processes properly.

```sh
sudo modprobe overlay
sudo modprobe br_netfilter
```

```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```sh
sudo sysctl --system
```

```sh
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

```sh
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

```sh
sudo systemctl restart containerd
```

#### Step 2: Kernel Parameter Configuration
This step ensures the kernel continues to allow bridged pod traffic to pass through iptables rules. These parameters are necessary because Kubernetes networking relies on bridge and iptables interactions for service discovery and pod-to-pod communication. The `sysctl --system` command reloads the values so they persist across reboots.

```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```sh
sudo sysctl --system
```

#### Step 3: Disable Swap and Configure Repo
Kubernetes requires swap to be disabled on the node because it can interfere with the scheduler and resource accounting. This step turns off swap for the current session and removes it from `/etc/fstab` so it stays disabled after reboot. Then we update the package index, install the required certificate and repository utilities, and add the official Kubernetes Debian repository for the stable v1.32 packages.

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```sh
sudo apt-get update
apt-cache madison kubeadm
sudo apt-get install -y kubelet=1.32.0-1.1 kubeadm=1.32.0-1.1 kubectl=1.32.0-1.1 cri-tools=1.32.0-1.1
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

#### Step 4 - Initialize Cluster with kubeadm:
This is the core step where the Kubernetes control plane is created. `kubeadm init` bootstraps the master node, creates the API server, scheduler, controller-manager, and etcd, and outputs a join command for worker nodes. The `--pod-network-cidr` defines the IP range for pods, and `--cri-socket` tells kubeadm to use containerd as the container runtime. After initialization, we copy the cluster admin kubeconfig so the current user can run `kubectl` commands on the new cluster.

```sh
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=1.32.0 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Step 5 - Remove the Taint:
By default, control-plane nodes are tainted so they do not run regular workload pods. This step removes that taint so the master node can also schedule application workloads in addition to managing the cluster. This is common in single-node or lab environments, but in production clusters you may keep the taint and add dedicated worker nodes.

```sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

#### Step 6 - Install Network Addon (Calico):
A Kubernetes cluster needs a Container Network Interface (CNI) plugin to provide pod networking. Calico creates virtual networks for pods, enables pod-to-pod communication, and supports network policies. The `kubectl create -f ...` commands apply the Calico operator and the default custom resource configuration, which installs the networking stack on the cluster.

```sh
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
```

#### Step 7 - Verification:
This final validation step confirms that the cluster is healthy and that pods can run successfully. `kubectl get nodes` checks node registration, `kubectl run nginx --image=nginx` creates a sample workload, and `kubectl get pods` verifies that the pod is scheduled and running. If these commands work, the cluster is functioning as expected.

```sh
kubectl get nodes
kubectl run nginx --image=nginx
kubectl get pods
```

#### Optional troubleshooting / reset
If the cluster setup fails or you want to start over, this cleanup command resets kubeadm state and removes any leftover Kubernetes networking and etcd data. It is useful when you need to reinitialize the cluster after a failed `kubeadm init` or configuration change.

```sh
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/etcd
```

Note: For a successful Kubernetes control-plane setup, ensure the host has a working DNS setup, the required ports are open, and swap is disabled before running `kubeadm init`.
