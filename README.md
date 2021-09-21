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

