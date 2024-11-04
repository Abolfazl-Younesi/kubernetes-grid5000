
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

### Example for Multi-Node Single Cluster

1. **Initialize the Control Plane (Master Node):**
   On the primary node, run:
   ```bash
   sudo-g5k kubeadm init --pod-network-cidr=10.244.0.0/16
   ```
   After initialization, Kubernetes will provide a `kubeadm join` command with a token and a hash, which allows worker nodes to join the cluster. The command looks like this:
   ```bash
   kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

2. **Where to Find the Token:**
   - If you lose the token or need to regenerate it, you can retrieve it on the control plane node:
     ```bash
     sudo-g5k kubeadm token list
     ```
   - To generate a new token, run:
     ```bash
     sudo-g5k kubeadm token create
     ```

   The discovery token CA certificate hash can be found with:
   ```bash
   openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
   openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
   ```

3. **Join Worker Nodes to the Cluster:**
   Once you have the token and hash, use the following format to join each worker node:
   ```bash
   sudo-g5k kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
   - **Example:**
     If your master node IP address is `192.168.1.10`, your token is `abcdef.0123456789abcdef`, and your hash is `1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef`, the command would be:
     ```bash
     sudo-g5k kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
     ```
   After running this command on each worker node, you can verify that the nodes have joined the cluster by running:
   ```bash
   kubectl get nodes
   ```

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
sudo-g5k firewall-cmd --zone=public --add-port=6443/tcp --permanent && \
sudo-g5k firewall-cmd --zone=public --add-port=10250/t

cp --permanent && \
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

This README now includes detailed instructions on where to find the token, as well as an example with placeholders (`master IP`, `token`, `hash`) explained.
