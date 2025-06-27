# Free5GC Installation(Single Cluster - multi node)

### **1. Prerequisites**
- Software
    - OS: Ubuntu 20.04.6 LTS
    - gcc 9.4.0
    - go 1.21.8 linux/amd64
    - kernel version 5.4.x-x-generic

| Host  name | vCPU | RAM | HDD | Network Interface|
| --- | --- | --- | --- | --- |
| free5gc-master | 4 vCPU | 8GB | 100GB | At least 1 Interface |
| free5gc-cp | 4 vCPU | 16GB | 100GB | 1 Interface |
| free5gc-up | 4 vCPU | 8GB | 100GB | 1 Interface |
| free5gc-an | 4 vCPU | 8GB | 100GB | 1 Interface |

### 2. Install

(1) Check Linux kernel version

```bash
# In Worker Nodes

uname -r

#sudo apt install linux-image-5.4.0-x-generic linux-headers-5.4.0-x-generic

#sudo sed -i "s/GRUB_DEFAULT=0/GRUB_DEFAULT='Advanced options for Ubuntu>Ubuntu, with Linux 5.4.0-169-generic'/g" /etc/default/grub

#sudo update-grub
#sudo reboot

#uname -r

```

![image.png](resources/image.png)

(2) Install gtp5g and gcc(9.4.0)

```bash
# In Worker Nodes

# install gtp5g(0.8.1 <= version < 0.9.0)
wget https://github.com/free5gc/gtp5g/archive/refs/tags/v0.8.10.tar.gz
tar xvfz v0.8.10.tar.gz
cd gtp5g-0.8.10/
sudo apt-get install gcc gcc-9 make -y

# compile gtp5g
sudo make
sudo make install
gcc -v
lsmod | grep gtp 
```

![image.png](resources/image%201.png)

(3) Install golang 1.21.8

```bash
# In All Nodes

# install golang 1.21.8
wget https://dl.google.com/go/go1.21.8.linux-amd64.tar.gz
sudo tar -C /usr/local -zxvf go1.21.8.linux-amd64.tar.gz
mkdir -p ~/go/{bin,pkg,src}

echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin:$GOROOT/bin' >> ~/.bashrc
echo 'export GO111MODULE=auto' >> ~/.bashrc
source ~/.bashrc

go version
```

![image.png](resources/image%202.png)

(4) Install Kubernetes and Flannel CNI

```bash
#In All Nodes

# update repo
sudo apt-get update

# add gpg key
sudo apt-get install -y curl ca-certificates gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# add docker repository
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# update the docker repo
sudo apt-get update

# install containerd
sudo apt-get install -y containerd.io

# set up the default config file
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
sudo systemctl restart containerd

# add the key for Kubernetes repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# add sources.list.d
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# update repo
sudo apt-get update

# enable ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1

# turn off swap filesystem
sudo swapoff -a

# install kubernetes
sudo apt-get install -y kubelet kubeadm kubectl

# exclude kubernetes packages from updates
sudo apt-mark hold kubelet kubeadm kubectl

# enable br_netfilter
sudo modprobe br_netfilter
sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables'# initialize kubeadm

```

(5) initialize kubeadm and cluster master node and worker nodes

```bash
# In Master Node

cd ~

# initialize kubeadm
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 | tee -a ~/k8s_init.log
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $USER:$USER $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" | tee -a ~/.bashrc
# remember token to cluster master node and worker nodes

kubectl taint nodes --all node-role.kubernetes.io/master-

# deploy cni(flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# deploy cni(multus)
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git && cd multus-cni
cat ./deployments/multus-daemonset.yml | kubectl apply -f -

kubectl get pod -A

```

![image.png](resources/image%203.png)

```bash
# In Worker Nodes

kubeadm join [Your_master_node_ip]:[port] --token ~~~~~~~~~~~~~~~~~ \
	--discovery-token-ca-cert-hash sha256: ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

![image.png](resources/image%204.png)

![image.png](resources/image%205.png)

![image.png](resources/image%206.png)

(6) Create namespace and install helm

```bash
# In Master Node

cd ~

# create namespace
kubectl create ns cp
kubectl create ns up
kubectl create ns an

# install helm3
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash ./get_helm.sh
rm ./get_helm.sh

helm version
```

![image.png](resources/image%207.png)

(7) Install docker

```bash
# In Master Node

# install docker 
sudo apt-get install -y docker-ce

# add user to docker group
sudo usermod -aG docker $USER

# bypass to run docker command
sudo chmod 666 /var/run/docker.sock

docker version
```

![image.png](resources/image%208.png)

(8) Create pv for mongodb

```bash
# In Master Node

cd ~

# create pv 
vim mongodb-pv.yaml

# change [USER] and [node name] in pv.yaml
sed -i "s/\[USER\]/$USER/g" ~/mongodb-pv.yaml
sed -i "s/\[cp-node-name\]/free5gc-cp/g" ~/mongodb-pv.yaml

# apply pv
kubectl apply -f mongodb-pv.yaml
kubectl get pv -A

# In Control Plane Node
cd ~
mkdir -p free5gc/mongodb/data-mongodb
```

```yaml
# mongodb-pv.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-mongodb
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/[USER]/free5gc/mongodb/data-mongodb # change here
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - [cp-node-name] # change here
```

![image.png](resources/image%209.png)

(9) Deploy free5gc chart with helm

Fork free5gc repository to your github account [https://github.com/Orange-OpenSource/towards5gs-helm](https://github.com/Orange-OpenSource/towards5gs-helm)

```bash
# In Master Nodes

# add helm repo
cd ~
mkdir 5g
cd 5g
helm repo add towards5gs 'https://raw.githubusercontent.com/Orange-OpenSource/towards5gs-helm/main/repo/'
helm repo update
helm search repo
helm pull towards5gs/free5gc; helm pull towards5gs/ueransim

tar -zxvf ueransim-2.0.17.tgz
tar -zxvf free5gc-1.1.7.tgz

# chage affinity to deploy pods to specific nodes
cd free5gc/charts

sed -i 's|affinity: {}|\
  affinity:\
    nodeAffinity:\
      requiredDuringSchedulingIgnoredDuringExecution:\
        nodeSelectorTerms:\
        - matchExpressions:\
          - key: kubernetes.io/hostname\
            operator: In\
            values: \
            - free5gc-cp|' free5gc-amf/values.yaml free5gc-ausf/values.yaml free5gc-dbpython/values.yaml free5gc-n3iwf/values.yaml free5gc-nrf/values.yaml free5gc-nssf/values.yaml free5gc-pcf/values.yaml free5gc-smf/values.yaml free5gc-udm/values.yaml free5gc-udr/values.yaml free5gc-webui/values.yaml 
   
sed -i 's|affinity: {}|\
  affinity:\
    nodeAffinity:\
      requiredDuringSchedulingIgnoredDuringExecution:\
        nodeSelectorTerms:\
        - matchExpressions:\
          - key: kubernetes.io/hostname\
            operator: In\
            values: \
            - free5gc-up|' free5gc-upf/values.yaml

sed -i 's|affinity: {}|\
  affinity:\
    nodeAffinity:\
      requiredDuringSchedulingIgnoredDuringExecution:\
        nodeSelectorTerms:\
        - matchExpressions:\
          - key: kubernetes.io/hostname\
            operator: In\
            values: \
            - free5gc-an|' ueransim/values.yaml 


# deploy upf
cd free5gc/charts
helm install userplane -n up \
--set global.n4network.masterIf=[up_node_network_interface] \
--set global.n3network.masterIf=[up_node_network_interface] \
--set global.n6network.masterIf=[up_node_network_interface] \
--set global.n6network.subnetIP="[up_node_subnet]" \
--set global.n6network.gatewayIP="[up_node_gateway]" \
--set upf.n6if.ipAddress="[up_node_ip]" \
free5gc-upf
```

![image.png](resources/image%2010.png)

![image.png](resources/image%2011.png)

```bash
# deploy control plane
cd ../../
helm upgrade --install controlplane -n cp \
--set deployUPF=false \
--set global.n2network.masterIf=[cp_node_network_interface] \
--set global.n3network.masterIf=[cp_node_network_interface] \
--set global.n4network.masterIf=[cp_node_network_interface] \
--set global.n6network.masterIf=[cp_node_network_interface] \
--set global.n9network.masterIf=[cp_node_network_interface] \
free5gc
```

![image.png](resources/image%2012.png)

![image.png](resources/image%2013.png)

![image.png](resources/image%2014.png)

![image.png](resources/image%2015.png)

http://[free5gc-cp node ip]:30500

Credentials: admin/free5gc

Register new subscriber

![image.png](resources/image%2016.png)

![image.png](resources/image%2017.png)

```bash
# deploy access network
helm upgrade --install ueransim -n an \
--set global.n2network.masterIf=[an_node_network_interface] \
--set global.n3network.masterIf=[an_node_network_interface] \
ueransim
```

![image.png](resources/image%2018.png)

![image.png](resources/image%2019.png)

![image.png](resources/image%2020.png)

![image.png](resources/image%2021.png)

![image.png](resources/image%2022.png)

```bash
# ue test 
kubectl exec -it -n an [ue-pod] -- bash

ip a

ping -I uesimtun0 8.8.8.8
```

![image.png](resources/image%2023.png)
