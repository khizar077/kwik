# Kubernetes with Kris (KwiK)
# Lesson 1: Building Kubernetes Live
Email: kris_applegate@dell.com
## Requirements to follow along in the lab:
- 4 Linux VMs (1 Master, 3 Worker)
- 4 vCPUs, 4GB memory, 40 GB OS Disk in each VM
- Linux OS of choice (CentOS 7.8 will be used as an example)
- IP / DNS Name resolution across all hosts
- 10 Network IPs unused on the same network as the VMs (for the LoadBalancer)

### Not Required (but recommended)
I like to do a couple other things just to make sure that the environment is hospitable to Kubernetes. Some are required and some are simple quality of life improvements:
1. Turn off SELinux (required)
- Edit `/etc/selinux/config` (change PERMISSIVE to disable)
2. Remove swap mount (required)
- Either run `swapoff` or by removing / commenting out the mount in `/etc/fstab` (and rebooting)
3. Install EPEL and open-vm-tools package (required in VMs)
``` 
yum install epel-release -y
yum install open-vm-tools -y
```
4. Password-less SSH setup from master node to all others
5. I'll also setup Clusterssh
```
yum install clustershell -y
```
- Configure groups in /etc/clustershell/groups.d/local.cfg (one for all and one for workers only)
6. Disable firewalld
```
systemctl disable firewalld
systemctl stop firewalld
```

## Container Runtime
1. Install Docker on each node (https://docs.docker.com/engine/install/centos/)
On **each** node perform the following:

```
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl enable docker 
systemctl start docker
```
2. Test Docker
```
docker run -it centos
hostname
exit
```
Verify that it gives you back a hostname different than the node you're on (should be a randomly generated one from Docker)

## Kubernetes tools
1. Now we need to install all the Kubernetes tools including:
- Kubeadm - Installation utility
- Kubelet - The heavy-lifting component of the Kubernetes runtime
- Kubectl - The CLI tool
2. Install the Kubernetes repo on each node (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- Run this on EVERY node:
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo yum install -y kubelet-1.17.12-0 kubeadm-1.17.12-0 kubectl-1.17.12-0 --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
- Note above that a specific version of Kubernetes is specified (1.17.12). You can use whichever version your like (omit any version for the latest/greatest). However, you should try and use a version that has broad support so the subsequent steps are likley to work as-is.

## Initialize your Kubernetes cluster
1. On the **master** only:
`kubeadm init --pod-network-cidr=192.168.0.0/16`
2. Copy the credentials to the cluster to your home directory so you can start interacting with the cluster:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3. Test
`kubectl get nodes`
- You should see your 1 master with a status of ready (may take a minute or two)
4. Copy your "join" string from the console to notepad (so we can use it to join the worker nodes to the cluster once the network overlay is installed)
	
## Network Overlay
This serves to unite Kubernetes nodes in an overlay allowing containers on each host to talk not just to themselves, but to each other.
1. Install Calico (https://docs.projectcalico.org/getting-started/kubernetes/quickstart)
```kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```
2. Check to make sure Calico comes online (3 services all have 1/1 containers running)
`watch kubectl get pods -n calico-system`
3. Join the workers to the cluster
- Run on **EACH** of the remaining nodes the "join" string you wrote down after the kubeadm init
- On the master check that you have 1 master and 3 workers (Role=<none>)
`kubectl get nodes` should look like the following:
```
[root@kube01 ~]# kubectl get nodes
NAME             STATUS   ROLES    AGE     VERSION
kube01.poc.csc   Ready    master   9m23s   v1.17.12
kube02.poc.csc   Ready    <none>   116s    v1.17.12
kube03.poc.csc   Ready    <none>   116s    v1.17.12
kube04.poc.csc   Ready    <none>   117s    v1.17.12

## Setup a Loadbalancer (MetalLB)
Now we need to be able to setup a workload to start serving our users. First we need to install something that can act as a LoadBalancer to all the containers we surface out of our cluster.
### Network Loadbalancer
1. Install Metallb: https://metallb.universe.tf/installation/
`kubectl edit configmap -n kube-system kube-proxy`
- Change the line about strictARP to true
- Exit with :wq
- Run the below:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
2. Now we need to create a config that lists what IPs we want to use for our LoadBalancer
- `vi metallb.yaml` (or use your text editor of choice)
- Paste in the below and change the IP range to the one you set aside according to the pre-requirements:
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```
- Replace in the above your range of IP addreeses you set aside in the pre-requirements secton for your LoadBalancer range.
3. Add the configuration we created above into kubernetes:
`kubectl create -f metallb.yaml`

## Test a Real Workload
Now that we have a working Kubernetes cluster along with a kubernetes LoadBalancer, lets run a demo application:
1. kubectl create -f https://github.com/mreferre/yelb/raw/master/deployments/platformdeployment/Kubernetes/yaml/yelb-k8s-loadbalancer.yaml
2. Check what IP your loadbalancer has given the yelb-ui service:
`kubectl get svc`
The output should look similar to:
```
[root@kube01 ~]# kubectl get svc
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes       ClusterIP      10.96.0.1       <none>         443/TCP        19m
redis-server     ClusterIP      10.98.154.170   <none>         6379/TCP       11s
yelb-appserver   ClusterIP      10.107.77.87    <none>         4567/TCP       11s
yelb-db          ClusterIP      10.102.47.112   <none>         5432/TCP       11s
yelb-ui          LoadBalancer   10.106.219.33   172.16.200.1   80:30506/TCP   11s
```
3. Hit that IP with a web browser to see if things worked!

## Next Steps
In the next KWiK lesson, we'll take this a couple steps further by adding in two kinds of storage services (NFS and Software-defined). After that, expect us to start going into the VMware Tanzu Kubernetes tools as well as a separate track focused around USING kubernetes with specific workloads (create an application, use data analytics tools, and more). 
