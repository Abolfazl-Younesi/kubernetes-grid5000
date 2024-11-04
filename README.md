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

## Adding Multiple Nodes to a Single Kubernetes Cluster

1. **Initialize the Control Plane (Master Node):**
   Run the following on the primary node:
   ```bash
   sudo-g5k kubeadm init
   ```
   This command outputs a `kubeadm join` command (similar to below) for adding worker nodes:
   ```bash
   kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

2. **Install Kubernetes on Each Worker Node:**
   Follow the above steps to install `kubelet`, `kubeadm`, `kubectl`, and `containerd` on each worker node, but **do not run `kubeadm init`** on these nodes.

3. **Join Worker Nodes to the Cluster:**
   Run the `kubeadm join` command on each worker node:
   ```bash
   sudo-g5k kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
   This command connects each worker node to the control plane, forming a multi-node cluster. Verify with:
   ```bash
   kubectl get nodes
   ```

## Setting Up Multiple Clusters

To set up multiple independent clusters, each with its own control plane and worker nodes:

1. **Initialize Each Control Plane Separately:**
   Run `kubeadm init` on a primary node for each cluster. Each control plane will have its own independent configuration.

2. **Manage Clusters with `kubectl` Contexts:**
   Use `kubectl` contexts to switch between clusters:
   ```bash
   kubectl config use-context <context-name>
   ```
   To list or add contexts, use:
   ```bash
   kubectl config get-contexts
   kubectl config set-context <context-name>
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
sudo-g5k firewall-cmd --zone=public --add-port=6443/tcp --permanent && \
sudo-g5k firewall-cmd --zone=public --add-port=10250/tcp --permanent && \
sudo-g5k firewall-cmd --zone=trusted --add-source=192.168.0.0/16 --permanent && \
sudo-g5k firewall-cmd --zone=trusted --add-source=10.140.0.0/16 --permanent && \
sudo-g5k firewall-cmd --reload
```

## Next Steps

After completing these steps, your system will be ready to join or initialize a Kubernetes cluster. To initialize a single-node cluster, run:

```bash
sudo-g5k kubeadm init
```

Refer to the Kubernetes documentation for more advanced cluster setups and configurations.

## Troubleshooting

- Ensure all commands are executed with `sudo-g5k`.
- Verify that each service (`kubelet`, `containerd`) is running with `sudo-g5k systemctl status <service>`.
- For further issues, consult Kubernetes documentation or community support.

## License

This guide is provided under the MIT License.
```

This README includes instructions for creating a multi-node single cluster and setting up multiple clusters, making it versatile for different Kubernetes setups.
