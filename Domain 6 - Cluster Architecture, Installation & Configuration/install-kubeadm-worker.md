# Install and Join a Kubernetes Worker Node with kubeadm

This guide covers the required steps to prepare a worker node and join it to an existing Kubernetes cluster.

## Step 1: Setup containerd

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
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```sh
sudo sysctl --system
```

```sh
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit the configuration file:

```sh
sudo nano /etc/containerd/config.toml
```

Set the following option to `true`:

```toml
SystemdCgroup = true
```

Then restart containerd:

```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Step 2: Kernel Parameter Configuration

```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```sh
sudo sysctl --system
```

## Step 3: Configure Repository and Install Kubernetes Components

```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

```sh
sudo install -m 0755 -d /etc/apt/keyrings
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

> Note: If a newer patch version is available, use the version shown by `apt-cache madison kubeadm` instead of hardcoding `1.32.0-1.1`.

## Step 4: Join the Worker Node

Run the exact `kubeadm join` command that was generated on the master/control-plane node. The command below is only an example.

```sh
sudo kubeadm join 209.38.120.248:6443 --token 9vxoc8.cji5a4o82sd6lkqa \
  --discovery-token-ca-cert-hash sha256:1818dc0a5bad05b378dd3dcec2c048fd798e8f6ff69b396db4f5352b63414baf
```

If your join token has expired, generate a new one on the control-plane node:

```sh
kubeadm token create --print-join-command
```

## Step 5: Verification (Run on the Master/Control Plane Node)

After the worker joins the cluster, verify that it is in `Ready` state:

```sh
kubectl get nodes
```

You should see the worker node listed as `Ready`.

Example:

```sh
NAME        STATUS   ROLES    AGE   VERSION
worker-1    Ready    <none>   2m    v1.32.0
```

## Troubleshooting

If the node is not ready, check the kubelet logs:

```sh
sudo journalctl -u kubelet -f
```

If `containerd` is failing, verify the configuration:

```sh
sudo containerd config dump
```
