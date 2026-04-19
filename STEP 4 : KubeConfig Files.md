## kubeConfig Files
A kubeconfig file is used by tools like kubectl and other clients to connect to the Kubernetes API server, containing cluster details, user credentials, and contexts.

In this section you will generate kubeconfig files for the controller manager, kube-proxy, scheduler clients and the admin user.
The controller manager and scheduler need to talk to the local API server, hence they use the localhost address.
Whereas in case of KubeProxy, for high availability, services running outside the control plane connect through the load balancer, so we first store its IP in a variable to use later in kubeconfig files for KubeProxy on worker nodes.






