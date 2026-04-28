## TLS Bootstrapping Worker Nodes
In the previous step we configured a worker node by

- Creating a set of key pairs for the worker node by ourself
- Getting them signed by the CA by ourself
- Creating a kube-config file using this certificate by ourself
- Everytime the certificate expires we must follow the same process of updating the certificate by ourself

This is not a practical approach when you could have 1000s of nodes in the cluster, and nodes dynamically being added and removed from the cluster. With TLS boostrapping:

- The Nodes can generate certificate key pairs by themselves
- The Nodes can generate certificate signing request by themselves
- The Nodes can submit the certificate signing request to the Kubernetes CA (Using the Certificates API)
- The Nodes can retrieve the signed certificate from the Kubernetes CA
- The Nodes can generate a kube-config file using this certificate by themselves
- The Nodes can start and join the cluster by themselves
- The Nodes can request new certificates via a CSR, but the CSR must be manually approved by a cluster administrator
