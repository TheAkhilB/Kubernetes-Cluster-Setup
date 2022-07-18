# Kubernetes Cluster Setup
## Before you begin
- A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps).
- 2 CPUs or more.
- Full network connectivity between all machines in the cluster (public or private network is fine).
- Unique hostname, MAC address, and product_uuid for every node. See here for more details.
- Certain ports are open on your machines. See here for more details.
- Swap disabled. You MUST disable swap in order for the kubelet to work properly.

## Kubeadm
Kubeadm is the “hard way” to begin with Kubernetes. With this solution, you will be able to bootstrap a minimum viable Kubernetes cluster that conforms to best practices. The cluster minimal size is composed of two nodes:
- Master node
- Worker node 
and you can add as many workers as you want.
But this solution is quite heavy to run on a laptop. Each node should be deployed in a VM and the minimal requirements are:
- Memory: 2 GB
- CPU: 2 (only for the master)

### Installing kubeadm, kubelet and kubectl
You will install these packages on all of your machines:
- kubeadm: the command to bootstrap the cluster.
- kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- kubectl: the command line util to talk to your cluster.
### Pre-Installation Steps On Both Master & Slave (To Install Kubernetes)
The following steps have to be executed on both the master and node machines. Let’s call the master as ‘kmaster‘ and the node as ‘knode‘. 
#### Step 1: 
#### Disable SWAP:
Kubernetes pods are designed to utilize CPU limits fully. The kubelet is not designed to use SWAP memory and it, therefore, needs to be disabled. Enter the following command in your terminal window to disable SWAP

`sudo swapoff -a`

After that you need to open the ‘fstab’ file and comment out the line which has mention of swap partition.

`sudo nano /etc/fstab`

#### Update The Hostnames
To change the hostname of both machines, run the below command to open the file and subsequently rename the master machine to ‘kmaster’ and your node machine to ‘knode’.

`sudo nano /etc/hostname`

Now go to the ‘hosts’ file on both the master and node and add an entry specifying their respective IP addresses along with their names ‘kmaster’ and ‘knode’. 

`sudo nano /etc/hosts`

#### Install OpenSSH-Server
Now we have to install openssh-server. Run the following command:

`sudo apt-get install openssh-server`

#### Step 2: 
Resolve nftables Backend Compatibility Issue: The current kubeadm packages are not compatible with the nftables backend. To avoid any issues, switch the iptables tooling to their legacy mode

`sudo update-alternatives --set iptables /usr/sbin/iptables-legacy`

`sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy`

#### Step 3:
#### Configure Firewall: 
Firewall settings need to be modified to allow communication among pods within a cluster. In this example, we edited the ports by using ufw

`sudo ufw allow 6443/tcp`

and also allow ports 2379,2380,10250,10251,10252,10255 respectively. (Check the official kubernetes Documentation if you are unaware of this)
#### Step 4:
#### Install Docker:    
A container runtime software manages container images on nodes. Without it, Kubernetes cannot access image libraries and execute the applications within the containers. For this purpose, we are going to install Docker. Install Docker on all the Master and Worker Nodes participating in your cluster. That means you need to repeat this process on each node in turn.
1. The following command updates Debian repositories and installs packages. It also allows your system to use repositories over a secure protocol, HTTPS:

`sudo apt-get update && sudo apt-get install apt-transport-https ca-certificates curl software-properties-common`

2. Add Docker’s official GPG key:

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`

3. Use the following command to set up the stable repository:

`echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

4. Install Docker Engine:

`sudo apt-get update`

`sudo apt-get install docker.io`

`sudo usermod -aG docker $USER `

5. Check the installation (and version) by entering the following:

`docker ––version`

6. Verify that Docker Engine is installed correctly by running the hello-world image.

`sudo docker run hello-world`

##### Start and Enable Docker
- Set Docker to launch at boot by entering the following:

`sudo systemctl enable docker`

- Verify Docker is running:

`sudo systemctl status docker`

- To start Docker if it’s not running:

`sudo systemctl start docker`

 7. Repeat the process on each server that will act as a node.


#### Step 5:
#### Change the cgroup-driver:
Make sure that both docker-ce and Kubernetes are using the same ‘cgroup’. Check the current docker cgroup:

`sudo docker info | grep -i cgroup`

Change the Docker cgroup-driver to ‘systemd’ if necessary by typing:

`sudo nano /etc/docker/daemon.json`

and enter the following,

`{
"exec-opts": ["native.cgroupdriver=systemd"]
}`

Save the file and reload the docker.

`sudo systemctl daemon-reload`

`sudo systemctl restart docker`

#### Step 6:
#### Install Kubernetes:
##### Add Official Signing Key:

`sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -;`

##### Add Repo:
Kubernetes is not included in the default repositories. To add them, enter the following in root user and exit from root

`echo 'deb http://packages.cloud.google.com/apt/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list;`

##### Install kubeadm, Kubelet And Kubectl :
Now it's time to install the 3 essential components. Kubelet is the lowest level component in Kubernetes. It’s responsible for what’s running on an individual machine. Kubeadm is used for administering the Kubernetes cluster. Kubectl is used for controlling the configurations on various nodes inside the cluster.

`sudo apt-get update`

`sudo apt-get install -y kubelet kubeadm kubectl`

#### Steps Only For Kubernetes Master (kmaster)
##### Initialize kubernetes with a Flannel compatible pod network CIDR :
We will now start our Kubernetes cluster from the master’s machine. Run the following command:

`sudo kubeadm init --pod-network-cidr=192.168.0.0/161`

Once this command finishes, it will display a kubeadm join message at the end. Make a note of the whole entry. This will be used to join the worker nodes to the cluster.
##### Setup kubectl:
As mentioned before, run the commands from the above output as a non-root user

`mkdir -p $HOME/.kube`

`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`

`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

##### Deploy Pod Network to Cluster:
A Pod Network is a way to allow communication between different nodes in the cluster. Here we are using Calico
Download the Calico networking manifest for the Kubernetes API datastore.

`curl https://docs.projectcalico.org/manifests/calico.yaml -O`

`sudo kubectl apply -f calico.yaml`

Verify that everything is running and communicating:

`kubectl get pods --all-namespaces`

#### Steps For Only Kubernetes Node VM (knode)

Follow Steps 1-6.3 in Slave node as well and do the following.

It is time to get your node, to join the cluster! This is probably the only step that you will be doing on the node, after installing kubernetes on it.
Run the join command that you saved, when you run ‘kubeadm init’ command on the master.
Join Worker Node to Cluster

As indicated in Step 6.4, you can enter the kubeadm join command on each worker node to connect it to the cluster.
Switch to the knode system and enter the command you noted from Step 6.4:

`sudo kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef 1.2.3.4:6443`

Replace the alphanumeric codes with those from your master server. Repeat for each worker node on the cluster. Wait a few minutes; then you can check the status of the nodes.
Switch to the master server, and enter:

`kubectl get nodes`

The system should display the worker nodes that you joined to the cluster.
##### In Master Node:
###### Install Kubernetes Dashboard:

`sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.0/aio/deploy/recommended.yaml`

###### Access Kubernetes Dashboard using Kubectl : 
Once we create the dashboard we can access it using Kubectl. To do this we will spin up a proxy server between our local machine and the Kubernetes apiserver.

`kubectl proxy`

We can access the Kubernetes dashboard UI by browsing to the following url:

`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

###### Kubernetes Dashboard Authentication:
For each of the following snippets for ServiceAccount and ClusterRoleBinding, you should copy them to new manifest files like dashboard-adminuser.yaml and use kubectl apply -f dashboard-adminuser.yaml to create them.
###### Creating a Service Account
We are creating a Service Account with the name admin-user in the namespace kubernetes-dashboard first.
This command will create a service account for dashboard in the kubernetes-dashboard namespace,

`sudo cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF`

###### Creating a ClusterRoleBinding
In most cases after provisioning a cluster using kops, kubeadm or any other popular tool, the ClusterRole cluster-admin already exists in the cluster. We can use it and create only ClusterRoleBinding for our ServiceAccount. If it does not exist then you need to create this role first and grant required privileges manually.
This command will add the cluster binding rules to your dashboard account,

`cat <<EOF | sudo kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF`

Getting a Bearer Token

Now we need to find a token we can use to log in. Execute following command:

`sudo kubectl describe secrets admin-user -n kubernetes-dashboard`

It should print something like:
ey******************************************************svnV6NYQ
Now copy the token and paste it into the Enter token field on the login screen.
Click the Sign in button and that's it. You are now logged in as an admin.
