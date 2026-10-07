#### Step 1: Setup containerd
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
```sh
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

#### Step 6 - Install Network Addon (Calico):
```sh
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
```

#### Step 7 - Verification:
```sh
kubectl get nodes
kubectl run nginx --image=nginx
kubectl get pods
```

#### Optional troubleshooting / reset
```sh
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/etcd
```

Note: For a successful Kubernetes control-plane setup, ensure the host has a working DNS setup, the required ports are open, and swap is disabled before running `kubeadm init`.
