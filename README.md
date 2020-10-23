# Documentation for Installing Prometheus and Grafana on Kubernetes

## Topics Covered
- [Creating Kubernetes Cluster using Kubeadm](README.md#creating-kubernetes-cluster-using-kubeadm)
  - [Prerequisites](README.md#prerequisites)
  - [Steps](README.md#steps)
- [Installing Prometheus, Grafana and Alertmanager](README.md#installing-prometheus-grafana-and-alertmanager)
  - [Install Helm](README.md#install-helm)
    - [Installing Helm Client](README.md#installing-helm-client)
    - [Setup Helm Server](README.md#setup-helm-server)
  - [Install Prometheus-Operator Helm Chart](README.md#install-prometheus-operator-helm-chart)
- [Setting up Nginx for Reverse proxy](README.md#setting-up-nginx-for-reverse-proxy)
  - [Steps](README.md#steps-1)
  - [Adding Basic Auth to Prometheus](README.md#adding-basic-auth-to-prometheus)
- [Configurations](README.md#configurations)
  - [Configure Prometheus](README.md#configure-prometheus)
  - [Configure Grafana](README.md#configure-grafana)
  - [Configure Alertmanager](README.md#configure-alertmanager)
    

## Creating Kubernetes Cluster using Kubeadm
This documentation shows how to create a Single Node Cluster on a **Ubuntu 18.04**. 

### Prerequisites
- 2 GB RAM (minimum).
- 2 CPU's (minimum).
- Swap should be disabled. See [here](README.md#disable-swap) for steps to disable swap.
- Full Network Connectivity between Nodes. See [here](README.md#deploy-calico-for-pod-network) for more details.

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
sudo systemctl status docker
```
Configure Docker to start on boot.
```
sudo systemctl enable docker
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

---

## Installing Prometheus, Grafana and Alertmanager
For this documentation we have used [Prometheus Opeartor](https://github.com/prometheus-operator/prometheus-operator) Helm chart to setup Prometheus, Grafana and Alertmanager in the Kubernetes Cluster.

### Install Helm
There are 2 parts for installing Helm:
- Helm Client ([Helm](README.md#installing-helm-client))
- Helm Server ([Tiller](README.md#setup-helm-server))

#### Installing Helm Client
The Helm version used for this documentation is v2.16.10. You can download the latest version as well.
- Download the desired [version](https://github.com/helm/helm/releases).
- Unpack the downloaded file. Replace the file name with the file you downloaded in the below command.
```
tar xvf helm-v2.16.10-linux-amd64.tar.gz
```
- Move the helm binary to the desired path.
```
mv linux-amd64/helm /usr/local/bin/helm
```
- Verify the installation by running `helm help` command.

#### Setup Helm Server
The easiest way to setup helm server is using `helm init`. After running `helm init`, check `helm version` if both client and server versions are shown, helm server has been setup successfully, if only Client Version is shown, follow the below steps to setup Tiller.

- Create specific account for tiller
```
kubectl --namespace kube-system create serviceaccount tiller
```

- Check if you have clusterrole
```
kubectl --namespace kube-system get clusterrole cluster-admin -o yaml
```

- Check if account "tiller" in first clause has a binding to clusterrole "cluster-admin" 
```
kubectl --namespace kube-system get clusterrolebinding
```

- Create new clusterrolebinding for tiller if not there already
```
kubectl --namespace kube-system create clusterrolebinding tiller-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

- Check if you really act as this account
```
kubectl --namespace kube-system get deploy tiller-deploy -o yaml
```

- The above output will not have settings "serviceAccount" and "serviceAccountName", so than add an account you want tiller to use
```
kubectl --namespace kube-system patch deploy tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

Now running `helm version` should show both Client and Server Versions.


### Install Prometheus-Operator Helm Chart
This chart bootstraps a prometheus-operator deployment on a Kubernetes cluster using the Helm. The Prometheus Operator uses Kubernetes CRD's (Custom Resource Definitions) to simplify the deployment and configuration of Prometheus, Grafana, Alertmanager, and other monitoring components.

#### Prerequisites
- Kubernetes 1.10+
- Helm 2.12+

To install the helm chart with name `prometheus-operator`, run the below command. Also you can use `--namespace <name_of_namespace>` in the command to specify any particular namespace. If not specified, chart will be installed in `default` namespace.
```
helm install --name prometheus-operator stable/prometheus-operator
```

---

## Setting up Nginx for Reverse proxy
We will be using Nginx as reverse proxy to serve Prometheus, Grafana and Alertmanager at separate sub-path's on the same port. Install Nginx from [here](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04).

#### Steps
- Go to the belwo path and open the file `default` in a file editor. 
```
cd /etc/nginx/sites-enabled/
sudo vi default
```
- Add the following content to the config file.
```
server {
        listen 9101 default_server;
        listen [::]:9101 default_server;

        # SSL configuration
        #
        # listen 443 ssl default_server;
        # listen [::]:443 ssl default_server;

        root /usr/share/nginx/html;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                proxy_pass http://localhost:30001/;
        }

        # Configuration to serve Grafana on /grafana sub-path
        location /grafana/ {
                proxy_pass http://localhost:30000/;
        }

        # Configuration to serve Prometheus on /prometheus sub-path
        location /prometheus/ {
                proxy_pass http://localhost:30002/;

                # Configuration to setup Basic Auth for Prometheus
                auth_basic "Prometheus";
                auth_basic_user_file /home/ubuntu/.htpasswd;
        }

        # Configuration to serve Alertmanager on /alertmanager sub-path
        location /alertmanager/ {
                proxy_pass http://localhost:30003/;
        }

}
```
### Adding Basic Auth to Prometheus
Prometheus by default doesn't provide Authentication, so to make Prometheus more secure, we use Prometheus in conjunction with a reverse proxy (Nginx in this case) and applying authentication at the proxy layer.
We'll use the htpasswd utility for this. This is in the apache2-utils packages.

- Install apache2-utils package.
```
sudo apt update
sudo apt install apache2-utils
```
- Create htpassword file by adding a new user. Provide the password when asked.
```
sudo htpasswd -c /home/ubuntu/.htpasswd admin
```
- Add these Basic Auth details in the Nginx Config file as shown above.
```
auth_basic "Prometheus";
auth_basic_user_file /home/ubuntu/.htpasswd;
```

---

## Configurations

After configuring Nginx for Reverse Proxy and Prometheus and other tools to be accessible on there resp sub-path's, now we have to configure each tool to serve on that sub-path.

#### Configure Prometheus
Prometheus Specs can be configured using the CRD Prometheus created by the Prometheus-Operator Helm Chart.
- Display the Prometheus resource. 
```
kubectl get prometheus
```
- Get the name of the service which Prometheus pod connects to.
```
kubectl get svc
```
- Open the `prometheus-operator-prometheus` Prometheus Resource in Text Editor.
```
kubectl edit prometheus prometheus-operator-prometheus
```
- Find the tag `externalUrl` in `spec` block and change its value as below. Replace `prometheus-operator-prometheus` with the name of service you got in 2nd point. Save and exit.
```
externalUrl: http://prometheus-operator-prometheus/prometheus/
```

Now edit the prometheus service to make it accessible to outside world.
```
kubectl edit svc prometheus-operator-prometheus
```
- Find the tag `type` and change its value to `NodePort`.
```
type: NodePort
```
- Specify node port number as below in the `ports` block. Save and exit.
```
nodePort: 30002
```

#### Configure Grafana
Grafana Configuration needs to be edited in a slighly different way then [Prometheus](README.md#configure-prometheus). Grafana Config is saved in a Config Map Resource, so we need to edit that Config Map in order to configure Grafana.
- Get the name of the Config Map.
```
kubectl get cm | grep grafana
```
- Edit the Config Map.
```
kubectl edit cm prometheus-operator-grafana
```
- Add the following lines in the `data` block. Save and exit.
```
[server]
  root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
  serve_from_sub_path = true
```

Now edit the grafana service to make it accessible to outside world.
- Get the name of the Grafana service.
```
kubectl get svc | grep grafana
```
- Edit the Service.
```
kubectl edit svc prometheus-operator-grafana
```
- Find the tag `type` and change its value to `NodePort`.
```
type: NodePort
```
- Specify node port number as below in the `ports` block. Save and exit.
```
nodePort: 30000
```

#### Configure Alertmanager
Alertmanager Specs can be configured using the CRD Alertmanager created by the Prometheus-Operator Helm Chart.
- Display the Alertmanager resource. 
```
kubectl get alertmanager
```
- Get the name of the service which Alertmanager pod connects to.
```
kubectl get svc | grep alertmanager
```
- Open the `prometheus-operator-alertmanager` Alertmanager Resource in Text Editor.
```
kubectl edit alertmanager prometheus-operator-prometheus
```
- Find the tag `externalUrl` in `spec` block and change its value as below. Replace `prometheus-operator-alertmanager` with the name of service you got in 2nd point. Save and exit.
```
externalUrl: http://prometheus-operator-alertmanager.default:9093/alertmanager
```

Now edit the alertmanager service to make it accessible to outside world.
```
kubectl edit svc prometheus-operator-alertmanager
```
- Find the tag `type` and change its value to `NodePort`.
```
type: NodePort
```
- Specify node port number as below in the `ports` block. Save and exit.
```
nodePort: 30003
```
