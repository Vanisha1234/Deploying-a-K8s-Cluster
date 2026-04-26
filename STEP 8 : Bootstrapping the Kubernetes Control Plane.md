## Bootstrapping the Kubernetes Control Plane

I will bootstrap the Kubernetes control plane across 2 compute instances and configure it for high availability. I will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

### Pre-requisites
The commands and the load balancer configuration must be run on each controller instance: controlplane01, and controlplane02.

### Provision the Kubernetes Control Plane
Download and Install the Kubernetes Controller Binaries
Download the latest official Kubernetes release binaries:
```bash
KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)

wget -q --show-progress --https-only --timestamping \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/${ARCH}/kube-apiserver" \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/${ARCH}/kube-controller-manager" \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/${ARCH}/kube-scheduler" \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/${ARCH}/kubectl"
```

Install the Kubernetes binaries:
```bash
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Configure the Kubernetes API Server
Place the key pairs into the kubernetes data directory and secure
```bash
{
  sudo mkdir -p /var/lib/kubernetes/pki

  # Only copy CA keys as we'll need them again for workers.
  sudo cp ca.crt ca.key /var/lib/kubernetes/pki
  for c in kube-apiserver service-account apiserver-kubelet-client etcd-server kube-scheduler kube-controller-manager
  do
    sudo mv "$c.crt" "$c.key" /var/lib/kubernetes/pki/
  done
  sudo chown root:root /var/lib/kubernetes/pki/*
  sudo chmod 600 /var/lib/kubernetes/pki/*
}
```
The instance internal IP address will be used to advertise the API Server to members of the cluster. The load balancer IP address will be used as the external endpoint to the API servers.
Retrieve these internal IP addresses:
```bash
LOADBALANCER=$(dig +short loadbalancer)
```
IP addresses of the two controlplane nodes, where the etcd servers are.
```bash
CONTROL01=$(dig +short controlplane01)
CONTROL02=$(dig +short controlplane02)
```
CIDR ranges used within the cluster
```bash
POD_CIDR=10.244.0.0/16
SERVICE_CIDR=10.96.0.0/16
```
Create the kube-apiserver.service systemd unit file:
```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${PRIMARY_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
  --enable-admission-plugins=NodeRestriction,ServiceAccount \\
  --enable-bootstrap-token-auth=true \\
  --etcd-cafile=/var/lib/kubernetes/pki/ca.crt \\
  --etcd-certfile=/var/lib/kubernetes/pki/etcd-server.crt \\
  --etcd-keyfile=/var/lib/kubernetes/pki/etcd-server.key \\
  --etcd-servers=https://${CONTROL01}:2379,https://${CONTROL02}:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/pki/ca.crt \\
  --kubelet-client-certificate=/var/lib/kubernetes/pki/apiserver-kubelet-client.crt \\
  --kubelet-client-key=/var/lib/kubernetes/pki/apiserver-kubelet-client.key \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/pki/service-account.crt \\
  --service-account-signing-key-file=/var/lib/kubernetes/pki/service-account.key \\
  --service-account-issuer=https://${LOADBALANCER}:6443 \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/pki/kube-apiserver.crt \\
  --tls-private-key-file=/var/lib/kubernetes/pki/kube-apiserver.key \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager
Move the kube-controller-manager kubeconfig into place:
```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```
Create the kube-controller-manager.service systemd unit file:
```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --allocate-node-cidrs=true \\
  --authentication-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --authorization-kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --bind-address=127.0.0.1 \\
  --client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
  --cluster-cidr=${POD_CIDR} \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/pki/ca.crt \\
  --cluster-signing-key-file=/var/lib/kubernetes/pki/ca.key \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --node-cidr-mask-size=24 \\
  --requestheader-client-ca-file=/var/lib/kubernetes/pki/ca.crt \\
  --root-ca-file=/var/lib/kubernetes/pki/ca.crt \\
  --service-account-private-key-file=/var/lib/kubernetes/pki/service-account.key \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler
Move the kube-scheduler kubeconfig into place:
```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```
Create the kube-scheduler.service systemd unit file:
```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --kubeconfig=/var/lib/kubernetes/kube-scheduler.kubeconfig \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Secure kubeconfigs
```bash
sudo chmod 600 /var/lib/kubernetes/*.kubeconfig
```

### Start the Controller Services
```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

### Verification
After running the above commands on both controlplane nodes, run the following on controlplane01
```bash
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```
> Output
```bash
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

> Remember to run the above commands on each controller node: controlplane01, and controlplane02.

## The Kubernetes Frontend Load Balancer
In this section you will provision an external load balancer to front the Kubernetes API Servers. The kubernetes-the-hard-way static IP address will be attached to the resulting load balancer.

### Provision a Network Load Balance
A NLB operates at layer 4 (TCP) meaning it passes the traffic straight through to the back end servers unfettered and does not interfere with the TLS process, leaving this to the Kube API servers.

Login to loadbalancer instance using vagrant ssh
```bash
sudo apt-get update && sudo apt-get install -y haproxy
```
Read IP addresses of controlplane nodes and this host to shell variables
```bash
CONTROL01=$(dig +short controlplane01)
CONTROL02=$(dig +short controlplane02)
LOADBALANCER=$(dig +short loadbalancer)
```
Create HAProxy configuration to listen on API server port on this host and distribute requests evently to the two controlplane nodes.

We configure it to operate as a layer 4 loadbalancer (using mode tcp), which means it forwards any traffic directly to the backends without doing anything like SSL offloading.
```bash
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
frontend kubernetes
    bind ${LOADBALANCER}:6443
    option tcplog
    mode tcp
    default_backend kubernetes-controlplane-nodes

backend kubernetes-controlplane-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server controlplane01 ${CONTROL01}:6443 check fall 3 rise 2
    server controlplane02 ${CONTROL02}:6443 check fall 3 rise 2
EOF
```
```bash
sudo systemctl restart haproxy
```

### Verification
Make a HTTP request for the Kubernetes version info:
```bash
curl -k https://${LOADBALANCER}:6443/version
```
> This should output some details about the version and build information of the API server.
```
{
  "major": "1",
  "minor": "35",
  "emulationMajor": "1",
  "emulationMinor": "35",
  "minCompatibilityMajor": "1",
  "minCompatibilityMinor": "34",
  "gitVersion": "v1.35.4",
  "gitCommit": "7b8c6cf0edd376b3d7c2f255142977c7f93db258",
  "gitTreeState": "clean",
  "buildDate": "2026-04-15T18:01:23Z",
  "goVersion": "go1.25.9",
  "compiler": "gc",
  "platform": "linux/amd64"
```
