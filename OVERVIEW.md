# Overview

This deploys a fork of Kubernetes the Hard Way (kthw) by [Mr. Kelsey Hightower](https://github.com/kelseyhightower). Mr. Hightower constantly updates kthw, so this repo is based on a snapshot in time, and in all likelihood kthw has vastly changed since I created this repository. Further, I have updated a few things and altered some others.

Here is what is used:

* Packet.net baremetal nodes (instead of GCP virtual machines)
* Mostly Ansible as opposed to command line
* Flannel as the networking plugin
* [Kubernetes](https://github.com/kubernetes/kubernetes) 1.10.7
* [containerd Container Runtime](https://github.com/containerd/containerd) 1.1.0
* [gVisor](https://github.com/google/gvisor) 08879266fef3a67fac1a77f1ea133c3ac75759dd
* [CNI Container Networking](https://github.com/containernetworking/cni) 0.6.0
* [etcd](https://github.com/coreos/etcd) 3.3.5

## Packet.net

I like Packet.net--getting baremetal resources is great. I specifically only use the `c2.medium.x86` instance size which is based on the AMD EPYC CPU. It also has 960GB of storage and 64GB of main memory for $1.00 an hour. I feel this is a good value.

Packet.net also has spot instances. I am often able to get instances for $0.30/hour in the Amsterdam location (I haven't tried other locations). That is a 70% discount. They do occasionally go away though; that is how spot instances work. For testing I love to use spot instances, but for production you would want on demand or even reserved instances. Not that I would use this repository in production. ;)

## Networking

Packet.net has an interesting layer 3 based networking model. Most people are used to a "network" being a /24 layer 2 based network, ala AWS VPC. However, that is not the only kind of networking, and, IMHO, layer 2 is something that should be avoided when possible. Packet.net uses L3 based networking in conjunction with some kind of filtering (aka network ACLs) to establish private networks for projects. Other nodes on the same project can access each other over this private network, but it is done at L3.

I decided to use Flannel because a VXLAN overlay makes a ton of sense with the underlying infrastructure being L3 based, and taking the filtering into consideration. I can't create my own arbitrary L3 networks and have Packet.net's infrastructure forward them. Thus VXLAN provided by Flannel becomes the overlay which is transported by Packet.net's physical L3 infrastructure. It's a nice combination.

In the future I'd like to move to a VXLAN infrastructure but one based on Open vSwitch or perhaps something like VPP.

*NOTE: Packet.net does support setting up VLANS. But I don't think you can do that through their API yet.*

## Architecture

Currently this deploys:

* 1 "controller" node that runs the kube-apiserver, kube-controller-manager and kube-scheduler
* 2 "worker" nodes that run kube-proxy, kubelet, containerd, fetch

With one controller it is not a highly available deployment. I have not tested three controllers yet, but it should work.

## Use of Ansible

I'm not (currently) using roles. Instead, because this is based on kthw, each step of the deployment is in its own standalone playbook, which are imported into `all.yml`. Not using roles was on purpose. In the future I may move to a role based model, but for now, it's a playbook based model.

## Security

The only major security consideration that has been made in this deployment is to try to ensure nothing but the k8s API is listening on the public IPs. Everything else should be listening on the private IPs only. Well, except whatever deployments you make public. :)

Otherwise no specific security considerations have been made.

## Why is this called Vent?

I didn't want to just call it "Kubernetes the Hard Way with Packet.net and Ansible" and I wanted to give it a name.
