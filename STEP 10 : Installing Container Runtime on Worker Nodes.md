## Installing Container Runtime on the Kubernetes Worker Nodes
Installed the Container Runtime Interface (CRI) on both worker nodes. CRI is a standard interface for the management of containers.
The commands must be run on each worker instance: node01, and node02

### Download and Install Container Networking
Here we will install the container runtime containerd from the Ubuntu distribution, and kubectl plus the CNI tools from the Kubernetes distribution. Kubectl is required on node02 to initialize kubeconfig files for the worker-node auto registration.

Update the apt package index and install packages needed to use the Kubernetes apt repository:
```bash
{
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
}
```
Set up the required kernel modules and make them persistent:
```bash
{
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

    sudo modprobe overlay
    sudo modprobe br_netfilter
}
```
Set the required kernel parameters and make them persistent:
```bash
{
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

    sudo sysctl --system
}
```
Determine latest version of Kubernetes and store in a shell variable:
```bash
KUBE_LATEST=$(curl -L -s https://dl.k8s.io/release/stable.txt | awk 'BEGIN { FS="." } { printf "%s.%s", $1, $2 }')
```
Download the Kubernetes public signing key:
```bash
{
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
}
```
Add the Kubernetes apt repository:
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBE_LATEST}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install the container runtime and CNI components:
```bash
sudo apt update
sudo apt-get install -y containerd kubernetes-cni kubectl ipvsadm ipset
```
Configure the container runtime to use systemd Cgroups, if not done results in a controlplane that comes up, then all the pods start crashlooping. kubectl will also fail with an error like ```The connection to the server x.x.x.x:6443 was refused - did you specify the right host or port?```
Create default configuration and pipe it through sed to correctly set Cgroup parameter:
```bash
{
    sudo mkdir -p /etc/containerd
    containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
}
```
Restart containerd
```bash
sudo systemctl restart containerd
```








