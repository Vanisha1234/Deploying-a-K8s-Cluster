Master Node Installation Design
<img width="1024" height="577" alt="image" src="https://github.com/user-attachments/assets/3a61b629-b268-4b66-aa8c-1c5feff24346" />

Firstly we will Deploy ETCD Cluster
Secondly, we will proceed with deploying Control plane components like API server, Scheduler and Controller
and Finally configure the Load Balancer. Here we will make use of HAProxy as load balancer configuration.
HAProxy will listen on port 6443 and split all traffic to the API Servers
All the nodes and other components that needs access to the api server will be pointed to the load balancer
Hence this way, if any of the master node get destroyed, the cluster will still stay active.
