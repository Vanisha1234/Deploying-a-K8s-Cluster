## Installing Client Tools
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
