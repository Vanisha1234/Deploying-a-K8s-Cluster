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

#### The Kubernetes API Server Certificate
The kube-apiserver certificate requires all names that various components may reach it to be part of the alternate names. These include the different DNS names, and IP addresses such as the controlplane servers IP address, the load balancers IP address, the kube-api service IP address etc. These provide an identity for the certificate, which is key in the SSL process for a server to prove who it is.
The openssl command cannot take alternate names as command line parameter. So we must create a conf file for it:
```bash
cat > openssl.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
IP.1 = ${API_SERVICE}
IP.2 = ${CONTROL01}
IP.3 = ${CONTROL02}
IP.4 = ${LOADBALANCER}
IP.5 = 127.0.0.1
EOF
```
Generate certs for kube-apiserver:
```bash
openssl genrsa -out kube-apiserver.key 2048

openssl req -new -key kube-apiserver.key \
  -subj "/CN=kube-apiserver/O=Kubernetes" -out kube-apiserver.csr -config openssl.cnf

openssl x509 -req -in kube-apiserver.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-apiserver.crt -extensions v3_req -extfile openssl.cnf -days 1000
```

#### The API Server Kubelet Client Certificate
This certificate is for the API server to authenticate with the kubelets when it requests information from them
```bash
cat > openssl-kubelet.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
basicConstraints = critical, CA:FALSE
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
```
Generate certs for kubelet authentication
```bash
{
  openssl genrsa -out apiserver-kubelet-client.key 2048

  openssl req -new -key apiserver-kubelet-client.key \
    -subj "/CN=kube-apiserver-kubelet-client/O=system:masters" -out apiserver-kubelet-client.csr -config openssl-kubelet.cnf

  openssl x509 -req -in apiserver-kubelet-client.csr \
    -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver-kubelet-client.crt -extensions v3_req -extfile openssl-kubelet.cnf -days 1000
}
```

#### The ETCD Server Certificate
Similarly ETCD server certificate must have addresses of all the servers part of the ETCD cluster. Similarly, this is a server certificate, which is again all about proving identity.

The openssl command cannot take alternate names as command line parameter. So we must create a conf file for it:
```bash
cat > openssl-etcd.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = ${CONTROL01}
IP.2 = ${CONTROL02}
IP.3 = 127.0.0.1
EOF
```
Generates certs for ETCD:
```bash
openssl genrsa -out etcd-server.key 2048

openssl req -new -key etcd-server.key \
  -subj "/CN=etcd-server/O=Kubernetes" -out etcd-server.csr -config openssl-etcd.cnf

openssl x509 -req -in etcd-server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out etcd-server.crt -extensions v3_req -extfile openssl-etcd.cnf -days 1000
```

#### The Service Account Key Pair
The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the managing service accounts documentation.

Generate the service-account certificate and private key:
```bash
openssl genrsa -out service-account.key 2048

openssl req -new -key service-account.key \
  -subj "/CN=service-accounts/O=Kubernetes" -out service-account.csr

openssl x509 -req -in service-account.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out service-account.crt -days 1000
```
