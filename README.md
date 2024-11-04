
# Kubernetes Setup Guide

This guide provides steps to set up Kubernetes on a Linux machine. It covers the installation of required packages, Kubernetes components, and container runtime, as well as necessary networking and firewall configurations.

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

## All-in-One Command

To streamline the setup process, you can execute the following combined command:

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

## Notes

- `kubelet`, `kubeadm`, and `kubectl` are core components for setting up and managing a Kubernetes cluster.
- `containerd` is used as the container runtime for managing containers in Kubernetes.
- The firewall configurations allow traffic necessary for Kubernetes node communication and access.
- Disabling swap is crucial for Kubernetes to function optimally.

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
