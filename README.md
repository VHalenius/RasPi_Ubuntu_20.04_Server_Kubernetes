# Installing Kubernetes on Raspberry Pi running on Ubuntu Server 20.04 LTS

This document describes how to install Kubernetes cluster on three Raspberry Pi 4B 8GB computers running Ubuntu Server 20.04 LTS. Ubuntu Server was installed using images written to SD cards using Raspberry Pi Imager.

## Changing host name

First step after starting up the Raspberry Pis was to change the hostname and set a static IP address.

Hostname is changed by updating the `/etc/hostname` file:

```
sudo nano /etc/hostname
```

First RasPi is named as `kubmaster` and the other two `kubnode1` and `kubnode2` respectively.

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
sudo apt upgrade -y
```

## Installing kubectl

```
sudo snap install kubectl --classic
```

## Installing container runtime

Kubernetes supports three container runtimes: Docker, containerd and CRI-O. In this document, Docker will be used. More information can be found in [Docker website](https://docs.docker.com/engine/install/ubuntu/#prerequisites) and [Kubernetes website](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

First task is to uninstall possible older Docker versions:
```
sudo apt remove docker docker-engine docker.io containerd runc
sudo apt update
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

### Configuring csgroup to use systemd:

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
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Install kubeadm

More information and pre-requisites can be found in [Kubernetes website](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

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



