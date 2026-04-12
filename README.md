# Deploying-a-K8s-Cluster

CLoning repo to local
repo link: https://github.com/mmumshad/kubernetes-the-hard-way.git

Prerequisites:
Installing VirtualBox
link: https://www.virtualbox.org/wiki/Downloads
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/205e9d96-4170-415d-bdab-c84ccfc2eba0" />
Installing Vagrant
link: https://www.vagrantup.com/

Provisioning Infrastructure for K8s and High Availability:
Provisioning infra on Virtual box VMs
Setup includes:
5 Node Setup(2 master, 2 worker nodes and 1 load balancer) with 1 vcpu each and 1 gb of memory each

Vagrantfile will be used to spin up this setup
It assigns the ip address to each one of them
Adds a DNS entry to each of the nodes to access the internet
Installs docker on the nodes

Navigate to directory where the vagrantfile is stored and run vagrant up command, this will automatically provision the setup.
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/f90e92f6-9011-450f-acab-1a10879d0bc7" />
<img width="1919" height="1009" alt="image" src="https://github.com/user-attachments/assets/a1508be5-fe36-422e-9d83-00d091e15a06" />


vagrant ssh <node-name> - to login to the node using vagrant command
<img width="926" height="697" alt="image" src="https://github.com/user-attachments/assets/3976398b-a6c9-4356-9c80-6d9ca07c5181" />

Using the private keys while logging in through any ssh terminal tool.
Once the nodes are created , a hidden <.vagrant> folder is created in the same folder as vagrant file
The .vagrant folder container directories related to all the files along with their private keys each.
<img width="861" height="361" alt="image" src="https://github.com/user-attachments/assets/191c048d-a175-43bf-a5d4-65d800fdd189" />
<img width="1170" height="363" alt="image" src="https://github.com/user-attachments/assets/ca9df9f6-1e73-45a6-9a94-3e741ee46b06" />
<img width="1113" height="574" alt="image" src="https://github.com/user-attachments/assets/583b057b-8ce8-4026-95d1-aaa2390911fb" />

Installing Client Tools
After provisioning the infrastructure, we need to select a administration client to perform administrative tasks like creating certificates, kubeconfig file and distributing them to other nodes etc.
In this case, we will be using Controlplane01
The administrative client selected must have ssh access to all the nodes in the cluster.
We will be setting up SSH based authentication from controlplane01 to other nodes.
We will be generating SSH key pair and will distribe the public key to other systems.
Generate SSH key pair on controlplane01 node:
```bash
ssh-keygen
```
Leave all settings to default by pressing ENTER at any prompt.
<img width="457" height="55" alt="image" src="https://github.com/user-attachments/assets/9a2434f3-2604-4c61-af59-e29bc92bb7d6" />

Copy the content of the public file, replace it in the following command and run the following command on all the nodes.
Add this key to the local authorized key(Controlplane01)
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
Moving the public key of the master to all the VMs:
cat >> ~/.ssh/authorized_keys <<EOF
Content of publick key file
EOF

Install Kubectl 
The kubectl command line utility is used to interact with the Kubernetes API Server.
Reference: https://kubernetes.io/docs/tasks/tools/
For Linux:
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/${ARCH}/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
Verify kubectl is installed:
kubectl version --client
