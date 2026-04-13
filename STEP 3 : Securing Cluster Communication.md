## Securing Cluster Communication
Provisioning a CA and Generating TLS Certificates
We will provision a PKI Infrastructure using the popular openssl tool, then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.
Will be performing these tasks on controlplane01 which is identified as administrative client in our case

### Certificate Authority
In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Query IPs of hosts we will insert as certificate subject alternative names (SANs), which will be read from /etc/hosts.
These commands fetch the IP addresses of your cluster nodes and load balancer, So you can add them into the API server certificate SAN list.
Kubernetes API server can be accessed from multiple endpoints, So the certificate must include all of these in SAN.
Set up environment variables, to avoid hardcoding IPs. Run the following:
CONTROL01=$(dig +short controlplane01)
CONTROL02=$(dig +short controlplane02)
LOADBALANCER=$(dig +short loadbalancer)

Determine the API server’s internal IP (the .1 address of the Service CIDR) and include it in the API server certificate as a SAN.
SAN (Subject Alternative Name) is a field in an SSL/TLS certificate that lists all the IP addresses and domain names the certificate is valid for.
SERVICE_CIDR=10.96.0.0/24
API_SERVICE=$(echo $SERVICE_CIDR | awk 'BEGIN {FS="."} ; { printf("%s.%s.%s.1", $1, $2, $3) }')

Check that the environment variables are set. Run the following:
echo $CONTROL01
echo $CONTROL02
echo $LOADBALANCER
echo $SERVICE_CIDR
echo $API_SERVICE
The output should look like this:
192.168.56.11
192.168.56.12
192.168.56.30
10.96.0.0/24
10.96.0.1
