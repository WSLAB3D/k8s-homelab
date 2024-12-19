# Kubernetes Cluster Setup: Common Issues and Solutions
When setting up a Kubernetes cluster, particularly in this vertualized environment, there are two critical issues that commonly occur and can prevent your cluster from functioning properly:

1. **Swap Memory Active**: Kubernetes requires swap to be disabled for consistent performance and resource management. This is a prerequisite that often gets overlooked.

2. **Container Runtime Misconfiguration**: Issues with containerd configuration and its interaction with Kubernetes, especially after fresh installations or updates.

These issues frequently appear after:
- Fresh installations
- System updates
- Moving VMs to different locations
- Changing system configurations

This guide provides a clear breakdown of how to identify and fix these issues, along with a comprehensive checklist for a successful Kubernetes cluster setup.

## 1. Pre-Installation Checklist
Before starting installation, verify:

- [ ] CPU virtualization enabled in BIOS
- [ ] Minimum 2 CPU cores allocated
- [ ] Minimum 8GB RAM
- [ ] Minimum 20GB disk space (40 is recommended)
- [ ] Ubuntu Server 22.04.5 LTS installed


## 2. Common Issues and Solutions


### 2.1 Swap Memory Issues

**Symptoms:**

Error in kubelet logs: "failed to run Kubelet: running with swap on is not supported"
Kubernetes components fail to start
kubeadm init fails with swap-related errors

**Solution:**
```bash
# Disable swap immediately
sudo swapoff -a

# Permanently disable swap
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled,Should show 0 for swap
free -h
```

### 2.2 Container Runtime Issues
Symptoms:

- Errors with crictl commands about "dockershim.sock"
- Multiple runtime endpoints errors
- API server connection refused
- Pods stuck in ContainerCreating state

Complete Solution:

```bash
# 1. Clean existing containerd configuration
sudo systemctl stop containerd
sudo rm -rf /etc/containerd/*
sudo rm -rf /var/lib/containerd/*

# 2. Create proper containerd configuration
cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.k8s.io/pause:3.9"
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
EOF

# 3. Configure crictl
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# 4. Restart services
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl restart kubelet
```


## 2.3 API Server Connection Issues
**Symptoms:**

kubectl get nodes returns "connection refused"
Unable to join worker nodes
Dashboard or other services can't connect to API

**Solution:**
```bash
# 1. Complete reset of Kubernetes
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d/*
sudo rm -rf $HOME/.kube/config
sudo rm -rf /var/lib/etcd/*

# 2. Reinitialize with explicit API server address
sudo kubeadm init --pod-network-cidr=172.16.0.0/12 --apiserver-advertise-address=$(hostname -I | cut -d' ' -f1)

# 3. Set up kubectl configuration
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# 3. Verification Steps
After fixing issues, verify:
```bash
# Check node status
kubectl get nodes

# Check all system pods
kubectl get pods -n kube-system

# Verify container runtime
sudo crictl ps

# Check service status
sudo systemctl status containerd
sudo systemctl status kubelet
```
# 4. When to Use These Solutions
Apply these fixes when:

-Setting up a new cluster
-After system updates
-After VM migration or cloning
-After network configuration changes
-When pods fail to start
- When worker nodes can't join

Order of Operations:

- Always fix swap issues first: as it's a prerequisite for kubelet to run
- Then address container runtime issues: as they affect how pods are managed
- Finally, handle Kubernetes configuration

Additional Tips:

- Backup and document any custom configurations and modifications
- Use kubectl describe for detailed problem investigation
- Check logs with journalctl -xeu kubelet for detailed errors
- Remember to apply network plugin (Calico) after cluster initialization
- Document IP addresses and ports used

# 5. Monitoring the Fix
After applying fixes, monitor:

- Node status (kubectl get nodes)
- Pod status (kubectl get pods -A)
- System logs (journalctl -xeu kubelet)
- Container runtime status (sudo crictl ps)