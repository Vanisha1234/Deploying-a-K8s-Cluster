<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/87fba332-3fd8-430d-ad2b-e74a7dca0aed" />

Two seperate approaches in configuring certificates
First Worker Node:
Generated certificates for ourselves
Get the certificate signed by CA, Copy it to the worker Node
Configure the Kubelet to use that certificate
Manual Renewal of certificate each time the certificate expires.

Second Worker Node:
Followed TLS Bootstrap approach
Configured the worker node to generate certificates, get it signed by CA, and start using it by itself
On expiration, the certificate will be renewed automatically by itself.

Configuration of Kube-proxy on both the nodes
