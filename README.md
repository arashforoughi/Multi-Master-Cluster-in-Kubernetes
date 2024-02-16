# Multi-Master-Cluster-in-Kubernetes

## Pre-requisite
For this demo, we will use 2 master and 2 worker node to create a multi master kubernetes cluster using kubeadm installation tool. Below are the pre-requisite requirements for the installation:

* 2 machines for master, ubuntu 22.04, 2 CPU, 2 GB RAM, 10 GB storage
* 2 machines for worker, ubuntu 22.04, 1 CPU, 2 GB RAM, 10 GB storage
* All machines must be accessible on the network. For cloud users - single VPC for all machines
* sudo privilege
* ssh access from master node to all machines (master & worker).
* ssh access can be given to any account. ssh through root is not mandatory

In this step we will install kubelet and kubeadm on the below nodes

* master1
* master2
* worker1
* worker2

The below steps will be performed on all the below nodes.
Log in to all the 4 machines as described above

### 1. Switch as root, update the repositories and disable swap on all the nodes:
```
# apt update
# swapoff -a 
# sed -i '/swap/s/^\//\#\//g' /etc/fstab
```

### 2. Install Containerd runtime on all nodes:
```
# apt install containerd.io
# wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
# mkdir -p /opt/cni/bin
# mv cni-plugins-linux-amd64-v1.1.1.tgz /opt/cni/bin/
# cd /opt/cni/bin; tar -zxvf cni-plugins-linux-amd64-v1.1.1.tgz
# containerd config default > /etc/containerd/config.toml
```

### 3. To use the systemd cgroup driver in containerd with runc:
```
# vi /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

# systemctl restart containerd
```

### 4. Install kubeadm, kubelet & kubectl on all nodes:
```
# sudo apt update; sudo apt install apt-transport-https ca-certificates curl -y
# sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
# echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
# sudo apt install kubeadm kubelet kubectl
```

### 5. Login to Master1 and initiate your kubernetes cluster:
The IP which has been wrote in front of the ```--control-plane-endpoint``` is the IP of master node interface which we needs our cluster published on it:
```
# kubeadm init --control-plane-endpoint "192.168.44.137:6443" --upload-certs --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address 192.168.44.137
```
your output should look like below:
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.44.137:6443 --token 6r0uef.3f1yycrzotyyny4i \
        --discovery-token-ca-cert-hash sha256:e57cbffd38f35ba6161f7e783d2198c14bb803d193a2bfcb7be36be9b0e979cc \
        --control-plane --certificate-key 70475e2d3e72765a13a1bacfd31ffe2cb905316abadeba6e1c0db4d0f7235900

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.44.137:6443 --token 6r0uef.3f1yycrzotyyny4i \
        --discovery-token-ca-cert-hash sha256:e57cbffd38f35ba6161f7e783d2198c14bb803d193a2bfcb7be36be9b0e979cc
```
The output consists of 3 major tasks:
  1. setup kube config using below for regular user on **master1**:
  ```
  $ mkdir -p $HOME/.kube
  $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
  or below for root user on **master1**:
  ```
  # export KUBECONFIG=/etc/kubernetes/admin.conf
  ```
  
  2. Setup new control plane (master) on **master2**:
  ```
  # kubeadm join 192.168.44.137:6443 --token 6r0uef.3f1yycrzotyyny4i \
        --discovery-token-ca-cert-hash sha256:e57cbffd38f35ba6161f7e783d2198c14bb803d193a2bfcb7be36be9b0e979cc \
        --control-plane --certificate-key 70475e2d3e72765a13a1bacfd31ffe2cb905316abadeba6e1c0db4d0f7235900
  ```
  
  3. Join worker nodes on **worker1** and **worker2**:
  ```
  # kubeadm join 192.168.44.137:6443 --token 6r0uef.3f1yycrzotyyny4i \
        --discovery-token-ca-cert-hash sha256:e57cbffd38f35ba6161f7e783d2198c14bb803d193a2bfcb7be36be9b0e979cc
  ```
 
  ***NOTE:***
  Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
  As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
  ```kubeadm init phase upload-certs --upload-certs``` to reload certs afterward.

for regenerating the join command for Master node use below commands:
```
# kubeadm init phase upload-certs --upload-certs
8801302dadf12348c59df41f990683095b191774bdd6750491015ff98a69fe98

# kubeadm token create --print-join-command
kubeadm join 192.168.10.100:6443 --token 354nd6.7tdroi4shmh6oasl --discovery-token-ca-cert-hash sha256:956c3a94edfdbd470873150213d4d68fe14bbb66ba52471be7f38a01a7c44026 -control-plane --certificate-key 8801302dadf12348c59df41f990683095b191774bdd6750491015ff98a69fe98
```

## Install CNI and complete installation
From one of the master nodes, run below command to deploy a pod network on cluster (here Flannel):
```
# kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

You can now verify your cluster using:
```
# kubectl get nodes
NAME       STATUS     ROLES                  AGE   VERSION
server-1   Ready      control-plane,master   33m   v1.24
server-2   Ready      control-plane,master   32m   v1.24
server-3   Ready      <none>                 32m   v1.24
server-4   Ready      <none>                 32m   v1.24
```
