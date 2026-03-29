# Kubernetes Cluster Setup Guide

This guide provides step-by-step instructions to set up a Kubernetes cluster on a Linux machine (e.g., Ubuntu). Follow the commands in order without modifying them.

## Prerequisites

- A Linux machine with sudo access.
- Internet connection for downloading packages and images.

## Step 1: Update System and Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

## Step 2: Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

## Step 3: Configure Sysctl Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Step 4: Install Containerd

```bash
sudo apt-get install -y containerd
```

## Step 5: Configure Containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
```

## Step 6: Restart Containerd

```bash
sudo systemctl restart containerd
```

## Step 7: Determine Kubernetes Version

```bash
KUBE_LATEST=$(curl -L -s https://dl.k8s.io/release/stable.txt | awk 'BEGIN { FS="." } { printf "%s.%s", $1, $2 }')
```

## Step 8: Add Kubernetes Repository

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

## Step 9: Install Kubernetes Components

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Step 10: Set Primary IP Address

Replace `<ip-address-of-the-machine>` with the actual IP address of your machine.

```bash
export PRIMARY_IP=<ip-address-of-the-machine>
```

## Step 11: Configure CRICTL

```bash
sudo crictl config \
    --set runtime-endpoint=unix:///run/containerd/containerd.sock \
    --set image-endpoint=unix:///run/containerd/containerd.sock
```

## Step 12: Set Pod and Service CIDRs

```bash
POD_CIDR=10.244.0.0/16
SERVICE_CIDR=10.96.0.0/16
```

## Step 13: Disable Swap and Reset Kubeadm

```bash
sudo swapoff -a
sudo kubeadm reset -f
sudo systemctl restart kubelet
sudo systemctl restart containerd
```

## Step 14: Remove Any Partially Initialized Kubeadm (if needed)

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/
sudo rm -rf $HOME/.kube
sudo systemctl restart containerd
sudo lsof -i :10250
sudo systemctl restart kubelet
```

## Step 15: Initialize the Kubernetes Cluster

```bash
sudo kubeadm init --pod-network-cidr $POD_CIDR --service-cidr $SERVICE_CIDR --apiserver-advertise-address $PRIMARY_IP
```

After running the above command, copy the `kubeadm join` command printed in the logs. Use it to join additional nodes to the cluster later.

## Step 16: Configure kubectl

```bash
mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
```

## Step 17: Verify Cluster Setup

```bash
kubectl get pods -n kube-system
```

## Step 18: Install Calico Network Plugin

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/tigera-operator.yaml
```

```bash
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.30.1/manifests/custom-resources.yaml
```

## Step 19: Update CIDR in Custom Resources

```bash
sed -i "s#192.168.0.0/16#$POD_CIDR#" custom-resources.yaml
```

```bash
kubectl apply -f custom-resources.yaml
```

## Step 20: Remove Control-Plane Taint (for single-node setup)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Step 21: Test the Cluster

```bash
kubectl create deployment nginx --image nginx:alpine
kubectl expose deploy nginx --type=NodePort --port 80
```

```bash
PORT_NUMBER=$(kubectl get service -l app=nginx -o jsonpath="{.items[0].spec.ports[0].nodePort}")
echo -e "\n\nService exposed on NodePort $PORT_NUMBER"
```

Replace `<vm-ip>` with the actual IP address of your VM.

```bash
curl http://<vm-ip>:$PORT_NUMBER
```