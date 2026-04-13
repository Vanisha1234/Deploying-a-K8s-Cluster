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

Create a CA certificate by first creating a private key, then using it to create a certificate signing request, then self-signing the new certificate with our key.
```bash
# Create private key for CA
openssl genrsa -out ca.key 2048

# Create CSR using the private key
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA/O=Kubernetes" -out ca.csr

# Self sign the csr using its own private key
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial -out ca.crt -days 1000
```
<img width="937" height="188" alt="image" src="https://github.com/user-attachments/assets/1eab688b-ac1f-4abe-9368-d3911a274483" />
The ca.crt is the Kubernetes Certificate Authority certificate and ca.key is the Kubernetes Certificate Authority private key, which will be used by the CA for signing certificates.

### Client and Server Certificates
We will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user.
#### The Admin Client Certificate
Generate the admin client certificate and private key:
```bash
# Generate private key for admin user
openssl genrsa -out admin.key 2048

# Generate CSR for admin user. Note the OU.
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr

# Sign certificate for admin user using CA servers private key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 1000
```
<img width="930" height="215" alt="image" src="https://github.com/user-attachments/assets/02932cde-13eb-4270-8b88-59a97dab0aab" />
Note that the admin user is part of the system:masters group. This is how we are able to perform any administrative operations on Kubernetes cluster using kubectl utility. The admin.crt and admin.key file gives you administrative access. We will configure these to be used with the kubectl tool to perform administrative functions on Kubernetes.


#### The Controller Manager Client Certificate
Generate the kube-controller-manager client certificate and private key:
```bash
openssl genrsa -out kube-controller-manager.key 2048

openssl req -new -key kube-controller-manager.key \
  -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" -out kube-controller-manager.csr

openssl x509 -req -in kube-controller-manager.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt -days 1000
```
<img width="847" height="131" alt="image" src="https://github.com/user-attachments/assets/561886bc-0b35-4f7c-a35f-ad2928f3dda7" />

#### The Kube Proxy Client Certificate
Generate the kube-proxy client certificate and private key:
```bash
openssl genrsa -out kube-proxy.key 2048

openssl req -new -key kube-proxy.key \
  -subj "/CN=system:kube-proxy/O=system:node-proxier" -out kube-proxy.csr

openssl x509 -req -in kube-proxy.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-proxy.crt -days 1000
```
<img width="974" height="152" alt="image" src="https://github.com/user-attachments/assets/3426d423-930d-4c06-bce2-1c8413fa2c5d" />

#### The Scheduler Client Certificate
Generate the kube-scheduler client certificate and private key:
```bash
openssl genrsa -out kube-scheduler.key 2048

openssl req -new -key kube-scheduler.key \
  -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" -out kube-scheduler.csr

openssl x509 -req -in kube-scheduler.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-scheduler.crt -days 1000
```
<img width="836" height="173" alt="image" src="https://github.com/user-attachments/assets/84e48b41-397a-431f-a47b-a3554557ef8a" />

