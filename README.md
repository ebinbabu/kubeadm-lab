# Setup Kubernetes Cluster

These steps are necessary to set up a Kubernetes cluster on both nodes. 

### Step 1: Enable iptables for bridge networking

Run the following commands on both nodes:

```bash
# Add necessary kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

# Configure iptables
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### Step 2: Install Kubernetes components

Install `kubeadm`, `kubectl`, and `kubelet` on all nodes:

```bash
# Update package list
sudo apt-get update

# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl

# Create a directory for apt keyrings
mkdir -p /etc/apt/keyrings

# Add Kubernetes apt key
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

# Add Kubernetes repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list again
sudo apt-get update

# Install specific versions of kubelet, kubeadm, and kubectl
sudo apt-get install -y kubelet=1.27.0-00 kubeadm=1.27.0-00 kubectl=1.27.0-00

# Prevent package updates for these components
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 3: Initialize the Kubernetes Control Plane (Master Node)

Run the following command on the control plane node to initialize the Kubernetes control plane:

```bash
# Get the IP address allocated to eth0
IP_ADDR=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

# Initialize the control plane node
kubeadm init --apiserver-cert-extra-sans=controlplane --apiserver-advertise-address $IP_ADDR --pod-network-cidr=10.244.0.0/16
```

### Step 4: Configure Kubeconfig for Control Plane

Once the initialization is done, set up the default kubeconfig file on the control plane node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Step 5: Deploy Pod Network Plugin

Run the following commands on the control plane node to deploy the Flannel network plugin:

```bash
# Download the Flannel YAML manifest
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml

# Open kube-flannel.yml with a text editor
# Find the args section within the kube-flannel container definition and add --iface=eth0 to the existing arguments list

# Apply the modified manifest
kubectl apply -f kube-flannel.yml
```

### Step 6: Join Worker Nodes to the Cluster

After the control plane initialization, you will receive a `kubeadm join` command with a token. SSH to the worker node and run the `kubeadm join` command as root:

```bash
# Example of the join command received from the control plane node
kubeadm join 192.23.108.8:6443 --token thubfg.2zq20f5ttooz27op --discovery-token-ca-cert-hash sha256:e7755ede5d6f0bae08bca4e13ccce8923995245ca68bba5f7ccc53c9b9728cdb
```

### Step 7: Verify the Cluster Status

Run the following command on the control plane node to check the status of the nodes:

```bash
kubectl get nodes
```

You should see both the control plane and worker nodes with a status of "Ready."

Your Kubernetes cluster is now set up and ready to use! For more information on deploying applications and other Kubernetes features, visit the official Kubernetes documentation. Happy clustering!