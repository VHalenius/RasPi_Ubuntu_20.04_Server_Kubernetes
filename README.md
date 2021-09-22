# Installing Kubernetes cluster on Raspberry Pis running on Ubuntu Server 20.04 LTS

This document describes how to install Kubernetes cluster on three Raspberry Pi 4B 8GB computers running Ubuntu Server 20.04 LTS. Ubuntu Server was installed using images written to SD cards using Raspberry Pi Imager.

Following steps are done in all nodes (control-plane(s) and worker nodes). Steps which are done only on some node types are mentioned specifically.

* [Changing host name](#changing-host-name)
* [Setting static IP-addresses](#setting-static-ip-addresses)
* [Setting correct time zone](#setting-correct-time-zone)
* [Updating packages](#updating-packages)
* [Installing container runtime](#installing-container-runtime)
* [Check pre-requisites](#check-pre-requisites)
* [Firewall rules](#firewall-rules)
* [Installing kubeadm, kubelet and kubectl](#installing-kubeadm-kubelet-and-kubectl)
* [Creating a single control-plane cluster with kubeadm](#creating-a-single-control-plane-cluster-with-kubeadm)
* [Installing pod network](#installing-pod-network)
* [Join worker nodes](#join-worker-nodes)
* [Installing local-path Persistent Storage provisioner](#installing-local-path-persistent-storage-provisioner)
* [Installing NFS client provisioner](#installing-nfs-client-provisioner)
* [Installing MetalLB load balancer](#installing-metallb-load-balancer)
* [Installing Helm](#installing-helm)


## Changing host name

First step after starting up the Raspberry Pis was to change the hostname and set a static IP address.

Hostname is changed by updating the `/etc/hostname` file:

```
sudo nano /etc/hostname
```

First RasPi (Control plane) is named as `kubmaster` and the other two (worker nodes) `kubnode1` and `kubnode2` respectively.

## Setting static IP-addresses

Next step is to set static IP-addresses. This can be done by editing the yaml-file in `/etc/netstat`. In my case the file name is `50-cloud-init.yaml`. It is a good idea to always make a backup of the file before editing. First RasPi, `kubmaster` will get and IP address of `192.168.0.230`, and `kubnode1` and `kubnode2` will get addresses `192.168.0.231` and `192.168.0.232` respectively. 

```
sudo nano /etc/netstat/50-cloud-init.yaml
```


```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.0.230/24
      gateway4: 192.168.0.254
      nameservers:
          addresses: [192.168.0.200]
```

After the changes, reboot and ssh into each system.

## Setting correct time zone

Default system time zone is Etc/UTC, and I needed to change it to Europe/Helsinki:

```
sudo unlink /etc/localtime
sudo ln -s /usr/share/zoneinfo/Europe/Helsinki /etc/localtime
```

## Updating packages

Updating the packages should probably be done as a first step, but I'm running the system as headless, so I made the baseic network related tasks first, so that I can ssh to each RasPi.

```
sudo apt update
```
```
sudo apt upgrade
```


## Installing container runtime

Kubernetes supports three container runtimes: Docker, containerd and CRI-O. In this document, Docker will be used. More information can be found in [Docker website](https://docs.docker.com/engine/install/ubuntu/#prerequisites) and [Kubernetes website](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

First task is to uninstall possible older Docker versions:
```
sudo apt remove docker docker-engine docker.io containerd runc
```
```
sudo apt update
```
```
sudo apt upgrade
```

### Set up the repository:
```
sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

Add Docker’s official GPG key:
```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Set up the stable repository:
```
echo \
  "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update packages
```
sudo apt update
```
```
sudo apt upgrade
```

### Install Docker engine:

```
sudo apt install docker-ce docker-ce-cli containerd.io
```

Verify that Docker engine is installed and working correctly:
```
sudo docker run hello-world
```

You should see something like this:
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
109db8fad215: Pull complete
Digest: sha256:61bd3cb6014296e214ff4c6407a5a7e7092dfa8eefdbbec539e133e97f63e09f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (arm64v8)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### Configuring cgroup to use systemd:

Next step is to configure csgroup to use systemd. More information can be found in [Kubernetes website](https://kubernetes.io/docs/setup/production-environment/container-runtimes/):

Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups:

```
sudo mkdir /etc/docker
```
```
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### Restart Docker and enable on boot

```
sudo systemctl enable docker
```
```
sudo systemctl daemon-reload
```
```
sudo systemctl restart docker
```

## Check pre-requisites

More information about pre-requisites can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

### Verify the MAC address and product_uuid are unique for every node

Checking MAC addresses:
```
ip link
```

The second rows for each adapter shows MAC addresses. For example, for adapter `eth0` MAC address is `e4:5f:01:27:d0:23`:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether e4:5f:01:27:d0:23 brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether e4:5f:01:27:d0:24 brd ff:ff:ff:ff:ff:ff
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:2e:88:56:35 brd ff:ff:ff:ff:ff:ff
```

Checking product_uuid:
```
sudo cat /etc/machine-id
```

### Letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```


```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```
sudo sysctl --system
```

Make sure that the `br_netfilter` module is loaded before this step. This can be done by running:
```
sudo lsmod | grep br_netfilter
```

You should see something like this:
```
br_netfilter           28672  0
bridge                225280  1 br_netfilter
```

### Ensure iptables tooling does not use the nftables backend

More information can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#ensure-iptables-tooling-does-not-use-the-nftables-backend)


Ensure legacy binaries are installed:
```
sudo apt-get install -y iptables arptables ebtables
```

Switch to legacy versions:
```
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
```
```
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```
```
sudo update-alternatives --set arptables /usr/sbin/arptables-legacy
```
```
sudo update-alternatives --set ebtables /usr/sbin/ebtables-legacy
```

### Enable cgroups

Add following to the end of `/boot/firmware/cmdline.txt`:
```
cgroup_enable=memory swapaccount=1 cgroup_memory=1 cgroup_enable=cpuset
```

> NOTE: add it to the end of the existing text, not in a new line

After updating `/boot/firmware/cmdline.txt` for all nodes, reboot the nodes:
```
sudo shutdown -r 0
```

## Firewall rules

Information about required ports can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports).

### Control plane nodes:

|Protocol|Direction|Port Range|Purpose|Used By|
|:---|:---|:---|:---|:---|
|TCP|Inbound|6443*|Kubernetes API server|All|
|TCP|Inbound|2379-2380|etcd server client API|kube-apiserver, etcd|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|10251|kube-scheduler|Self|
|TCP|Inbound|10252|kube-controller-manager|Self|

### Setting firewall rules for Control plane:

Since this is a new install, and firewall is not enabled yet, all incoming traffic is set to denied by default, and allowen on case-by-case basis:
```
sudo ufw default deny incoming
```

Allow SSH:
```
sudo ufw allow 22 comment 'SSH'
```

Allow HTTP:
```
sudo ufw allow 80 comment 'HTTP'
```

Allow Kubernetes API server:
```
sudo ufw allow 6443/tcp comment 'Kubernetes API server'
```

Allow etcd server client API:
```
sudo ufw allow 2379:2380/tcp comment 'etcd server client API'
```

Allow Kubelet API:
```
sudo ufw allow 10250/tcp comment 'Kubelet API'
```

Allow kube-scheduler:
```
sudo ufw allow 10251/tcp comment 'kube-scheduler'
```

Allow kube-controller-manager:
```
sudo ufw allow 10252/tcp comment 'kube-controller-manager'
```

Enable firewall:
```
sudo ufw enable
```

### Worker node(s):

|Protocol|Direction|Port Range|Purpose|Used By|
|:---|:---|:---|:---|:---|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|30000-32767|NodePort|Services†	All|

† Default port range for [NodePort Services](https://v1-17.docs.kubernetes.io/docs/concepts/services-networking/service/).

Any port numbers marked with * are overridable, so you will need to ensure any custom ports you provide are also open.

Although etcd ports are included in control-plane nodes, you can also host your own etcd cluster externally or on custom ports.

### Setting firewall rules for worker nodes:

Denying all incoming traffic by default
```
sudo ufw default deny incoming
```

Allow SSH:
```
sudo ufw allow 22 comment 'SSH'
```

Allow Kubelet API:
```
sudo ufw allow 10250/tcp comment 'Kubelet API'
````

Allow NodePort:
```
sudo ufw allow 30000:32767/tcp comment 'NodePort'
```

Enable firewall:
```
sudo ufw enable
```


## Installing kubeadm, kubelet and kubectl

More information can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt update
```
```
sudo apt install -y kubelet kubeadm kubectl
```
```
sudo apt-mark hold kubelet kubeadm kubectl
```



## Creating a single control-plane cluster with kubeadm

> NOTE: Control plane only!

More information can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

Control plane is initialized using `kubeadm init`. Apiserver advertise address is used to set the advertise address for this particular control-plane node’s API server. Pod network cidr specifies the network in which pods are created. In my case I'm using Control plane's (`kubmaster`) IP-address as apiserver address, and `10.0.0.0/8` as pod network cidr:

```
sudo kubeadm init --apiserver-advertise-address=192.168.0.230 --pod-network-cidr=10.0.0.0/8
```

You should see following text indicating that the installation was successful. Note the commands to run for starting to use the cluster, as well as the command to join the nodes:
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.230:6443 --token qp324u.5spbiugteb7i7iyr \
        --discovery-token-ca-cert-hash sha256:b708db7f05ec100fa7049e562909944fa9a6aefb563e4cf5b1aae6aff3cb3bb3
```


### Connect kubectl to cluster:

```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Test kubectl:

```
kubectl get nodes
```

It might take a while, but you should see the following:
```
NAME        STATUS   ROLES                  AGE   VERSION
kubmaster   Ready    control-plane,master   12m   v1.22.2
```


## Installing pod network

In this document Calico is used as pod network. Mode information can be found in [Calico website](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises).

```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```
```
kubectl apply -f calico.yaml
```

## Join worker nodes

> NOTE: worker nodes only

Enter following commands in worker nodes. Use the token which was given to you in step "Creating a single control-plane cluster with kubeadm":
```
sudo kubeadm join 192.168.0.230:6443 --token yh2ofi.2xx3bcs66xbc68f3 \
        --discovery-token-ca-cert-hash sha256:faf6a5ec00fb44d91d23ef4b55b414f1cfb37d892428b0b24cbae4129c9a300d
```
You should see following information:

```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

When running `kubectl get nodes` in Control plan, you should see the following:
```
NAME        STATUS   ROLES                  AGE     VERSION
kubmaster   Ready    control-plane,master   35m     v1.22.2
kubnode1    Ready    <none>                 2m16s   v1.22.2
kubnode2    Ready    <none>                 63s     v1.22.2
```

If you want to give more descriptive role to workers you can add labels in the Control plane node:
```
kubectl label node kubnode1 node-role.kubernetes.io/worker=worker
```
```
kubectl label node kubnode2 node-role.kubernetes.io/worker=worker
```

Now when you enter `kubectl get nodes` in the Control plane node, you will get:
```
NAME        STATUS   ROLES                  AGE     VERSION
kubmaster   Ready    control-plane,master   38m     v1.22.2
kubnode1    Ready    worker                 5m38s   v1.22.2
kubnode2    Ready    worker                 4m25s   v1.22.2
```

## Installing local-path Persistent Storage provisioner

>NOTE: Control plane only

For automatically allocating disk space for persistent storage, a Persistent Storage provisioner is needed. In this document `local-path` provisioner from Rancher is used. More information can be found in [Rancher Git](https://github.com/rancher/local-path-provisioner/blob/master/README.md#usage).

Installing the provisioner:
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

You should see the following:
```
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
configmap/local-path-config created
```


Setting local-path provisioner as default:
```
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

You should see the following:
```
storageclass.storage.k8s.io/local-path patched
```
## Installing NFS client provisioner

Useful [Youtube video](https://www.youtube.com/watch?v=DF3v2P8ENEg)  from "Just me and Opensource"

```
sudo ufw allow 2049 comment 'NFS'
```

```
sudo ufw allow 111 comment 'NFS'
```

```
sudo apt install rpcbind nfs-kernel-server
```

```
sudo mkdir -p /srv/nfs/kubedata
```

```
sudo chown nobody: /srv/nfs/kubedata
```

```
sudo nano /etc/exports
```

Add row:
```
/srv/nfs/kubedata *(rw,sync,no_root_squash,no_subtree_check,no_all_squash,insecure)
```

```
sudo systemctl enable --now nfs-server
```

```
sudo exportfs -rav
```

```
sudo showmount -e localhost
```

### Clients (worker nodes)

```
sudo ufw allow 2049 comment 'NFS'
```

```
sudo ufw allow 111 comment 'NFS'
```

```
sudo apt update
sudo apt-get install rpcbind nfs-common
```

```
sudo nano /etc/hosts.deny
```

Add line:
```
rpcbind : ALL
```

```
sudo nano /etc/hosts.allow
```

Add line:
```
rpcbind :  YOUR_NFS_SERVER_IP_ADDRESS_HERE
# In my case
rpcbind : 192.168.0.230
```

### Install the NFS client provisioner

>NOTE: Control plane only

```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```
```
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path
```

>NOTE! Below installation instruction does not work

Download yaml files from:
```
https://github.com/justmeandopensource/kubernetes/tree/master/yamls/nfs-provisioner
```

Change `deployment.yaml` file, and update NFS server IP address, and change image pointing to arm-version:
```
image.repository=quay.io/external_storage/nfs-client-provisioner-arm
```

```
kubectl create -f default-sc.yaml
```

```
kubectl create -f rbac.yaml
```

```
kubectl create -f deployment.yaml
```

## Installing MetalLB load balancer
>NOTE: Control plane only

More information can be found in [MetalLB website](https://metallb.universe.tf/installation/). Youtube channel "Just me and Opencource" has a great [video](https://www.youtube.com/watch?v=2SmYjj-GFnE) on installing and configuring MetalLB. 

### Firewall rules

Allow traffic (TCP and UDP) between nodes. Apply firewille rule to all nodes (Control plane(s) and workers):
```
sudo ufw allow 7946 comment 'MetalLB'
```

### Check kube-proxy mode

To see if kube-proxy is running on IPVS mode:
```
kubectl -n kube-system describe cm kube-proxy | grep mode
```

If you see `mode: ""` then kube-broxy is not running on IPVS-mode, and strict ARP mode does not need to be enabled. In other cases, check the [Preparation](https://metallb.universe.tf/installation/#preparation) section in MetalLB website.

### Installing MetalLB

To install MetalLB, apply the manifest. First one creates a namespace for MetlLB:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
```

You should see:
```
namespace/metallb-system created
```

To install the load balancer:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

You should see:
```
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/controller created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
role.rbac.authorization.k8s.io/pod-lister created
role.rbac.authorization.k8s.io/controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
rolebinding.rbac.authorization.k8s.io/pod-lister created
rolebinding.rbac.authorization.k8s.io/controller created
daemonset.apps/speaker created
deployment.apps/controller created
```

### Configuration

I am using Layer 2 configuration i.e. using a specific IP address range for load balacer to give out addresses to services. You can check more about [configuration options](https://metallb.universe.tf/configuration/#layer-2-configuration) in MetalLB website.

First, create a temporary config-file, for example `temp.yaml` in your home directory. Then paste the configuration in the file, and change the `addresses:` -section to match the address range you want to use. I'm using ten addresses `192.168.0.240-192.168.0.249` in my subnet.
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
      - 192.168.0.240-192.168.0.249
```

Then apply the configuration:
```
kubectl create -f temp.yaml
```

You can test to see if the load balancer works by installing nginx, and creating a load balancer service for it:
```
kubectl create deploy nginx --image nginx
```

Creating the load balancer service:
```
kubectl expose deploy nginx --port 80 --type LoadBalancer
```

Service should have received an external IP address from the range you specified:
```
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>          443/TCP        42m
nginx        LoadBalancer   10.109.1.154   192.168.0.240   80:31728/TCP   3m46s
```





## Installing Helm

>NOTE: Control plane only

More information can be found in [Helm website](https://helm.sh/docs/intro/install/)

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
```

```
sudo apt install apt-transport-https --yes
```

```
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```

```
sudo apt update
```

```
sudo apt install helm
```

## Plex Media Server

Use `values.yaml` from this git repo. Change plex claim [token](https://plex.tv/claim)
```
helm install plex k8s-at-home/plex -f values.yaml
```

```
kubectl expose deploy plex --port 32400 --type LoadBalancer --name plexlb
```

## CIFS Flexvolume Plugin for Kubernetes

Run in all nodes (Control plane(s) and nodes)
```
VOLUME_PLUGIN_DIR="/usr/libexec/kubernetes/kubelet-plugins/volume/exec"
```

```
sudo mkdir -p "$VOLUME_PLUGIN_DIR/fstab~cifs"
```

```
cd "$VOLUME_PLUGIN_DIR/fstab~cifs"
```

```
sudo curl -L -O https://raw.githubusercontent.com/fstab/cifs/master/cifs
```

```
sudo chmod 755 cifs
```

Install packages:
```
sudo apt install cifs-utils
```

```
sudo apt install jq
```

```
