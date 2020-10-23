# Documentation for Installing Prometheus and Grafana on Kubernetes

## Creating Kubernetes Cluster using Kubeadm
This documentation shows how to create a Single Node Cluster on a **Ubuntu 18.04**. 

### Prerequisites
- 2 GB RAM (minimum)
- 2 CPU's (minimum)
- Swap should be disabled. See [here](README.md#disable-swap) for steps to disable swap
- Full Network Connectivity between Nodes. See [here](README.md#deploy-calico-for-pod-network) for more details

### Steps

##### Disable Swap
```
swapoff -a
```
##### Configuring iptables to see bridged traffic
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```
#### Install Docker
```
sudo apt-get update
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker
```
#### Install Kubeadm, Kubelet and Kubectl
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```
#### Initialize Master Node
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
#### Deploy Calico for Pod Network
We will be using Calico for Network Connectivity. You can download these `yaml` files and configure them acc to your needs.
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```
#### Untaint Master Node
By default Master Node has taints added so that pods can not be scheduled on it. To allow Kubernetes to schedule pods on Master Node, run the below command.
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
#### Confirm Kubernetes Pods are Up and Running
After setting up the Pod Network, you can verify that everything is up and running. Once the CoreDNS pod starts running, you can go ahead with further installations.
```
kubectl get pods --all-namespaces
```
