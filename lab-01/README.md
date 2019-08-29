# Lab 01 - Installation

Minikube is the preferred installation, if however you cannot use Minikube you 
can alternatively install Kubernetes on your cloud VM using kubeadm.

## Minikube

For these labs we will use minikube as our Kubernetes environment. This is
going to run locally on your host.

Minikube is a tool that makes it easy to run Kubernetes locally. Minikube runs a
single-node Kubernetes cluster inside a VM on your laptop for users looking to
try out Kubernetes or develop with it day-to-day.

### Task 0: Installing VirtualBox

By default `minikube` uses `VirtualBox` to run its VM.  To install VirtualBox
download the latest [package](https://download.virtualbox.org/virtualbox/6.0.4/VirtualBox-6.0.4-128413-OSX.dmg)
and install it.


### Task 1: Installing minikube

The easiest way to install minikube is with [homebrew](https://brew.sh/). The
following command will install minikube.

```
brew cask install minikube
```

### Task 2: Confirming minikube installation

To verify that the installation is succesfull we can try to get the version of
minikube with the following command :

```
minikube version

minikube version: v0.35.0
```

### Task 3: Running minikube

To start minikube run the `minikube start` command (it might take a couple of minutes
before minikube has started):

```
minikube start

😄  minikube v0.35.0 on darwin (amd64)
💡  Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
🏃  Re-using the currently running virtualbox VM for "minikube" ...
⌛  Waiting for SSH access ...
📶  "minikube" IP address is 192.168.99.102
🐳  Configuring Docker as the container runtime ...
✨  Preparing Kubernetes environment ...
🚜  Pulling images required by Kubernetes v1.13.4 ...
🔄  Relaunching Kubernetes v1.13.4 using kubeadm ...
⌛  Waiting for pods: apiserver proxy etcd scheduler controller addon-manager dns
📯  Updating kube-proxy configuration ...
🤔  Verifying component health ......
💗  kubectl is now configured to use "minikube"
🏄  Done! Thank you for using minikube!
```

### Task 4: Checking minikube status

To verify that minikube is running correctly run the `minikube status` command:

```
minikube status

host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.102
```

## Kubeadm

If it's not possible to install Minikube, we will install kubeadm on our VM 
running in the cloud.

### Task 1: Installing docker

Kubernetes needs a docker runtime installed on our vm:

```
curl -fsSL https://get.docker.com/ | sh
```

### Task 2: Installing prerequisites

Next login as root user and install kubeadm aswell as the the Kubernetes components:

```
sudo su

apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### Task 3: Initialising kubeadm

Now we initialise a single host Kubernetes cluster and configure our kubectl:

```
kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Task 4: Installing network add-on

Afterwards we configure our pod networking via the Calico add-on

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

### Task 5: Checking status

Now we wait untill our networking is fully operational, if all pods are on the 
running status your cluster is good to go:

```
kubectl get pods --all-namespaces

NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-7bd78b474d-6jzq8    1/1     Running   0          34m
kube-system   calico-node-4txlc                           1/1     Running   0          34m
kube-system   coredns-5c98db65d4-cqjxv                    1/1     Running   0          35m
kube-system   coredns-5c98db65d4-kz78p                    1/1     Running   0          35m
kube-system   etcd-kubernetes-cursus                      1/1     Running   0          34m
kube-system   kube-apiserver-kubernetes-cursus            1/1     Running   0          34m
kube-system   kube-controller-manager-kubernetes-cursus   1/1     Running   0          34m
kube-system   kube-proxy-sh7h4                            1/1     Running   0          35m
kube-system   kube-scheduler-kubernetes-cursus            1/1     Running   0          34m
```
### Task 6: Untainting node

By default , your cluster will not schedule pods on the control-plane node for security reasons. If you want to be able to schedule pods on the control-plane node, e.g. for a single-machine Kubernetes cluster for development, run:

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```