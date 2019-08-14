---
layout: post
title:  "kubic-control and own kubernetes control plan on openSUSE Kubic"
date:   2019-08-14 08:54:00 +0200
author: Thorsten Kukuk
---
## Why should I want to use kubic-control?

We have [kubeadm]({{ site.baseurl }}{% post_url 2018-08-20-kubeadm-intro %}) on openSUSE Kubic to manage kubernetes, why do we need yet another management tool?
There is a nice blog giving an answer to this [Automated High Availability in kubeadm v1.15: Batteries Included But Swappable](https://kubernetes.io/blog/2019/06/24/automated-high-availability-in-kubeadm-v1.15-batteries-included-but-swappable/), which explains, beside kubernetes multi master, also the scope of kubeadm very well: "kubeadm is focused on performing the actions necessary to get a minimum viable, secure cluster up and running in a user-friendly way. kubeadm’s scope is limited to the local machine’s filesystem and the Kubernetes API, and it is intended to be a _composable building block for higher-level tools_."

The following tasks are out of scope:
- Infrastructure provisioning
- Third-party networking
- Non-critical add-ons, e.g. monitoring, logging and visualitzation
- Specific cloud provider integrations

kubic-control is such a higher-level toolkit. It configures the Host OS (in the feature, it will even be able to install new baremetal nodes with help of [yomi](https://github.com/openSUSE/yomi)), setups all necessary services, uses kubeadm to deploy kubernetes and installs and configures all necessary add-ons, like pod network and update and reboot services for the OS and the cluster. It will also help you to avoid mistakes like not calling kubeadm with the right arguments if you want to use flannel for the network.

## What is kubic-control?

kubic-control consists of three binaries:
- kubicd, a daemon which communicates via gRPC with clients. It's setting up kubernetes on openSUSE Kubic, including pod network, kured, transactional-update, ...
- kubicctl, a cli interface
- haproxycfg, a cli interface adjust haproxy.cfg for use as loadbalancer for the kubernetes API

The communication is encrypted, the kubicctl command can run on any
machine. The user authenticates with his certificate, using RBAC to determine
if the user is allowed to call this function. kubiccd will use kubeadm and
kubectl to deploy and manage the cluster. So the admin can at everytime modify
the cluster with this commands, too, there is no hidden state-database except
for the informations necessary for a kubernetes multi-master/HA setup.

## Requirementes

Mainly generic requirements by kubernetes itself:

- All the nodes on the cluster must be on a the same network and be able to communicate directly with each other.
- All nodes in the cluster must be assigned static IP addresses. Using dynamically assigned IPs will break cluster functionality if the IP address changes.
- The Kubernetes master node(s) must have valid Fully-Qualified Domain Names (FQDNs), which can be resolved both by all other nodes and from other networks which need to access the cluster.
- Since Kubernetes mainly works with certificates and tokens, the time on all Nodes needs to be always in sync. Else communication inside the cluster will break.

As salt is used for the communication, the Admin Node needs to run a
salt-master and all other nodes needs to be configured as salt-minion.

## Installation

Currently, the easiest way to get a running kubernetes cluster on openSUSE
Kubic is to take the [DVD](http://download.opensuse.org/tumbleweed/iso/) and
install the nodes with YaST. There are three relevant system roles to choose
from:
- Kubic Admin Node
- Additional Kubic Node
- Loadbalancer

### Kubic Admin Node

The Kubic Admin Node is running _kubicd_, the _salt-master_ and the first kubernetes
master node. The overhead for _kubicd_ and the _salt-master_ is very low, so
you don't need a bigger machine because of this.
During the first boot, some certificates are created for _kubicd_ in _/etc/kubicd/pki`:

- Kubic-Control-CA.key - the private CA key
- Kubic-Control-CA.crt - the public CA key. This one is needed by kubicctl, too
- KubicD.key - the private key for kubicd
- kubicD.crt - the signed public key for kubicd
- admin.key - private key, allows kubicctl to connect to kubicd as admin
- admin.crt - public key, allows kubicctl to connect to kubcd as admin

For _kubicctl_, you need to create a directory _~/.config/kubicctl_ which contains _Kubic-Control-CA.crt_, _user.key_ and _user.crt_. For the admin role, this need to be a copy of admin.key and admin.crt. For other users, you need to create corresponding certificates and sign them with _Kubic-Control-CA.crt_. If you call _kubicctl_ as root and there is no _user.crt_ in _~/.config/kubicctl_, the admin certificates from _/etc/kubicd/pki_ are used if they exist. Certificates for additional users can be created with _kubicctl certificates create <account>_.

Please take care of this certificates and store them secure, this are the passwords to access kubicd!


### Additional Kubic Node

Additional Kubic Nodes will become either additional master nodes for HA of
the kubernetes API, or worker nodes. The procedure is in both cases the same:
- Install the Additional Kubic Node system role
- Configure salt-minion: `echo "master: <FQHN of salt-master/Kubic Admin
Node>" > /etc/salt/minion.d/master.conf`
- Start salt with _systemctl enable --now salt-minion_
- Accept the salt-minion on the Kubic Admin Node with _salt-key -A_

### Kubic Loadbalancer Node

The Kubic Loadbalancer Node system role installs a MicroOS without container runtime but haproxy instead. This system role is currently under development and not yet fully functional. After installation, the salt-minion needs to be configured:
- Configure salt-minion: `echo "master: <FQHN of salt-master/Kubic Admin
Node>" > /etc/salt/minion.d/master.conf`
- Start salt with _systemctl enable --now salt-minion_
- Accept the salt-minion on the Kubic Admin Node with _salt-key -A_

Afterwards, the haproxy needs to configured and enabled. In one of the next
versions, kubic-control should be able to do this itself.

## Next Steps

This is just the beginning of our journey to make kubic-control a tool to
manage your whole openSUSE Kubic cluster. Many more things are planned or
under active development:

* Re-configure a haproxy loadbalancer when adding or removing master nodes
* Install new nodes with yomi
* Certificate handling
* Extend support and handling of add-ons, including rolling updates
* Enhance YaST2 to configure the salt-minion already during installation
* Integration of ignition

As with any new software, especially stuff we're changing so quickly, there is a chance of bugs. If you try the steps in this guide and find any, please report them to our [Bugzilla](http://bugzilla.opensuse.org/enter_bug.cgi?product=openSUSE+Tumbleweed&format=guided) under the "Kubic" component.

**Please Note:**  kubic-control on Kubic is under heavy active development. Many new functionality will come which may change current  functionality.

Thanks, and have a lot of fun!
