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

### Step 2: Download and Install Container Networking

The commands in this lab must be run on each instance: `control plane`, and `worker`. Login to each instance using SSH Terminal.

The versions chosen here align with those that are installed by the current kubernetes-cni package for a v1.24 cluster.

```bash
CONTAINERD_VERSION=1.7.2
CNI_VERSION=1.3.0
RUNC_VERSION=1.1.7

wget -q --show-progress --https-only --timestamping \
  https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz \
  https://github.com/containernetworking/plugins/releases/download/v${CNI_VERSION}/cni-plugins-linux-amd64-v${CNI_VERSION}.tgz \
  https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64

sudo mkdir -p /opt/cni/bin

sudo chmod +x runc.amd64
sudo mv runc.amd64 /usr/local/bin/runc

sudo tar -xzvf containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz -C /usr/local
sudo tar -xzvf cni-plugins-linux-amd64-v${CNI_VERSION}.tgz -C /opt/cni/bin
```

Next, create the containerd service unit.

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF

sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock --set image-endpoint=unix:///run/containerd/containerd.sock &&

Now start it

```bash
sudo systemctl enable containerd
sudo systemctl start containerd
```

### Step 3: Install Kubernetes components

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

### Step 4: Initialize the Kubernetes Control Plane (Master Node)

Run the following command on the control plane node to initialize the Kubernetes control plane:

```bash
# Get the IP address allocated to enp1s0 | ensure that's the correct interface name

IP_ADDR=$(ip addr show enp1s0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

# Initialize the control plane node
kubeadm init --apiserver-cert-extra-sans=controlplane --apiserver-advertise-address $IP_ADDR --pod-network-cidr=10.244.0.0/16
```

Output:
```
[init] Using Kubernetes version: v1.27.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W0527 08:20:20.776881   17295 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.2, falling back to the nearest etcd version (3.5.7-0)
W0527 08:20:29.514363   17295 checks.go:835] detected that the sandbox image "k8s.gcr.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [controlplane kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.23.108.8]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key


[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [controlplane localhost] and IPs [192.23.108.8 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [controlplane localhost] and IPs [192.23.108.8 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
W0527 08:20:33.516573   17295 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.2, falling back to the nearest etcd version (3.5.7-0)
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 12.002756 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node controlplane as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node controlplane as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: udz7gw.ock7lve14hw0cdpp
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

kubeadm join 192.23.108.8:6443 --token udz7gw.ock7lve14hw0cdpp \
        --discovery-token-ca-cert-hash sha256:e7755ede5d6f0bae08bca4e13ccce8923995245ca68bba5f7ccc53c9b9728cdb
```

### Step 5: Configure Kubeconfig for Control Plane

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

### Step 8: Deploy Pod Network Plugin

Run the following commands on the control plane node to deploy the Flannel network plugin:

```bash
# Download the Flannel YAML manifest
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml

# Open kube-flannel.yml with a text editor
# Find the args section within the kube-flannel container definition It should look like this
args:
  - --ip-masq
  - --kube-subnet-mgr
#Add the additional argument - --iface=eth0 to the existing list of arguments
# Apply the modified manifest
kubectl apply -f kube-flannel.yml
```

You should see both the control plane and worker nodes with a status of "Ready."

Your Kubernetes cluster is now set up and ready to use! For more information on

 deploying applications and other Kubernetes features, visit the official Kubernetes documentation. Happy clustering!