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

Two ways of logging into the nodes
1. vagrant ssh <node-name> - to login to the node using vagrant command
<img width="926" height="697" alt="image" src="https://github.com/user-attachments/assets/3976398b-a6c9-4356-9c80-6d9ca07c5181" />

2. Using the private keys while logging in through any ssh terminal tool.
Once the nodes are created , a hidden <.vagrant> folder is created in the same folder as vagrant file
The .vagrant folder container directories related to all the files along with their private keys each.
<img width="861" height="361" alt="image" src="https://github.com/user-attachments/assets/191c048d-a175-43bf-a5d4-65d800fdd189" />
<img width="1170" height="363" alt="image" src="https://github.com/user-attachments/assets/ca9df9f6-1e73-45a6-9a94-3e741ee46b06" />
<img width="1113" height="574" alt="image" src="https://github.com/user-attachments/assets/583b057b-8ce8-4026-95d1-aaa2390911fb" />



