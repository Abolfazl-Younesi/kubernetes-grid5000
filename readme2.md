
# Kubernetes Setup Guide

This guide provides steps to set up Kubernetes on a Linux machine, including installation of required packages, Kubernetes components, and container runtime. It also covers networking and firewall configurations, adding multiple nodes to a cluster, and managing multiple clusters.

## Prerequisites

- Linux machine with `sudo-g5k` access
- Internet connection for package downloads

## Installation Steps

### 1. Install Required Packages

```bash
sudo-g5k apt-get update
sudo-g5k apt-get install -y apt-transport-https ca-certificates curl gpg
```

### 2. Set Up Keyrings Directory

```bash
sudo-g5k mkdir -p -m 755 /etc/apt/keyrings
```

### 3. Add Kubernetes GPG Key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo-g5k gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 4. Add Kubernetes Repository

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo-g5k tee /etc/apt/sources.list.d/kubernetes.list
```

### 5. Update Package List

```bash
sudo-g5k apt-get update
```

### 6. Install Kubernetes Components and Container Runtime

```bash
sudo-g5k apt-get install -y kubelet kubeadm kubectl containerd
```

### 7. Configure `containerd`

```bash
sudo-g5k mkdir /etc/containerd/
containerd config default | sudo-g5k tee /etc/containerd/config.toml
sudo-g5k sed -i 's/^#?\sSystemdCgroup\s=.*$/SystemdCgroup = true/' /etc/containerd/config.toml
```

### 8. Start and Enable Services

```bash
sudo-g5k systemctl start containerd
sudo-g5k systemctl enable --now kubelet
```

### 9. Enable Networking for Kubernetes

```bash
sudo-g5k modprobe br_netfilter
sudo-g5k sysctl -w net.ipv4.ip_forward=1
```

### 10. Disable Swap

```bash
sudo-g5k swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

### 11. Configure Firewall Settings

```bash
sudo-g5k firewall-cmd --zone=public --add-port=6443/tcp --permanent
sudo-g5k firewall-cmd --zone=public --add-port=10250/tcp --permanent
sudo-g5k firewall-cmd --zone=trusted --add-source=192.168.0.0/16 --permanent
sudo-g5k firewall-cmd --zone=trusted --add-source=10.140.0.0/16 --permanent
sudo-g5k firewall-cmd --reload
```

## Configuring the Master Node

### Initialize the Control Plane

1. Run the following command on the master node to initialize the control plane:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   ```

2. Set up local `kubectl` configuration:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. **Optional:** Transfer the `admin.conf` file to worker nodes if needed (for example, using `scp`) before it gets removed.

### Configure Flannel Network Add-On

1. Download the Flannel configuration file:
   ```bash
   wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

2. Update the Flannel network configuration:
   - Open the `kube-flannel.yml` file for editing:
     ```bash
     vim kube-flannel.yml
     ```
   - Locate `"Network": "10.244.0.0/16"` and replace it with `"Network": "192.168.0.0/16"` to match the pod network CIDR specified during `kubeadm init`.

3. Apply the Flannel network configuration:
   ```bash
   kubectl apply -f kube-flannel.yml
   ```

### Generate the Join Command for Worker Nodes

Run the following command on the master node to generate the join command for worker nodes:
```bash
sudo kubeadm token create --print-join-command
```

The output will look something like this:
```bash
kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

Use this command on each worker node to join it to the cluster.

## Adding Multiple Nodes to a Single Kubernetes Cluster

1. Follow the steps above to initialize the master node and generate the join command.
2. Install Kubernetes on each worker node following the installation steps but **do not run `kubeadm init`** on the worker nodes.
3. Run the join command on each worker node:
   ```bash
   sudo-g5k kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
   - Replace `<master-ip>`, `<token>`, and `<hash>` with the values generated in the join command on the master node.

## Setting Up Multiple Clusters

### Example for Multiple Independent Clusters

To create two independent clusters, repeat the following steps for each primary node to act as a control plane for a separate cluster.

1. **Initialize Each Control Plane Separately:**
   Run `kubeadm init` on each cluster’s main node with different network configurations to avoid IP conflicts:
   ```bash
   sudo-g5k kubeadm init --pod-network-cidr=10.244.0.0/16  # for Cluster 1
   sudo-g5k kubeadm init --pod-network-cidr=10.245.0.0/16  # for Cluster 2
   ```
   Each `kubeadm init` command generates a unique `kubeadm join` command specific to its cluster.

2. **Manage Clusters with `kubectl` Contexts:**
   Use `kubectl` contexts to switch between clusters. For example:
   ```bash
   # Add Cluster 1
   kubectl config set-context cluster1 --cluster=cluster1 --user=user1 --namespace=default
   kubectl config use-context cluster1

   # Add Cluster 2
   kubectl config set-context cluster2 --cluster=cluster2 --user=user2 --namespace=default
   kubectl config use-context cluster2
   ```

3. **Verify Cluster Contexts:**
   Use the following command to list all contexts:
   ```bash
   kubectl config get-contexts
   ```
   Switching between clusters is done by specifying the desired context:
   ```bash
   kubectl config use-context cluster1  # Switch to Cluster 1
   kubectl config use-context cluster2  # Switch to Cluster 2
   ```

## All-in-One Command for Single Cluster Setup

To streamline the setup process, use the following combined command for a single-cluster configuration:

```bash
sudo-g5k apt-get update && sudo-g5k apt-get install -y apt-transport-https ca-certificates curl gpg && \
sudo-g5k mkdir -p -m 755 /etc/apt/keyrings && \
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo-g5k gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg && \
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo-g5k tee /etc/apt/sources.list.d/kubernetes.list && \
sudo-g5k apt-get update && \
sudo-g5k apt-get install -y kubelet kubeadm kubectl containerd && \
sudo-g5k mkdir /etc/containerd/ && \
containerd config default | sudo-g5k tee /etc/containerd/config.toml && \
sudo-g5k sed -i 's/^#?\sSystemdCgroup\s=.*$/SystemdCgroup = true/' /etc/containerd/config.toml && \
sudo-g5k systemctl start containerd && \
sudo-g5k systemctl enable --now kubelet && \
sudo-g5k modprobe br_netfilter && \
sudo-g5k sysctl -w net.ipv4.ip_forward=1 && \
sudo-g5k swapoff -a && \
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true && \
sudo-g5k firewall-cmd --zone=public --add-port=6443/tcp --per

manent && \
sudo-g5k firewall-cmd --zone=public --add-port=10250/tcp --permanent && \
sudo-g5k firewall-cmd --zone=trusted --add-source=192.168.0.0/16 --permanent && \
sudo-g5k firewall-cmd --zone=trusted --add-source=10.140.0.0/16 --permanent && \
sudo-g5k firewall-cmd --reload
```

## License

This guide is provided under the MIT License.
```

This README now includes complete master node configuration, including the network add-on setup and generating the join command for worker nodes.
