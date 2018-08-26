# Kubernetes the Hard Way on Packet.net with Ansible

A couple of years ago I worked on [Kubernetes the Hard Way on AWS with Ansible](https://github.com/ccollicutt/kubernetes-the-hard-way-with-aws-and-ansible) and now I'm doing it again, but this time with Packet.net and Ansible, and, of course, a much updated kthw.

## Overview

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

### Packet.net

I like Packet.net--getting baremetal resources is great. I specifically only use the `c2.medium.x86` instance size which is based on the AMD EPYC CPU. It also has 960GB of storage and 64GB of main memory for $1.00 an hour. I feel this is a good value.

Packet.net also has spot instances. I am often able to get instances for $0.30/hour in the Amsterdam location (I haven't tried other locations). That is a 70% discount. They do occasionally go away though; that is how spot instances work. For testing I love to use spot instances, but for production you would want on demand or even reserved instances. Not that I would use this repository in production. ;)

### Networking

Packet.net has an interesting layer 3 based networking model. Most people are used to a "network" being a /24 layer 2 based network, ala AWS VPC. However, that is not the only kind of networking, and, IMHO, layer 2 is something that should be avoided when possible. Packet.net uses L3 based networking in conjunction with some kind of filtering (aka network ACLs) to establish private networks for projects. Other nodes on the same project can access each other over this private network, but it is done at L3.

I decided to use Flannel because a VXLAN overlay makes a ton of sense with the underlying infrastructure being L3 based, and taking the filtering into consideration. I can't create my own arbitrary L3 networks and have Packet.net's infrastructure forward them. Thus VXLAN provided by Flannel becomes the overlay which is transported by Packet.net's physical L3 infrastructure. It's a nice combination.

In the future I'd like to move to a VXLAN infrastructure but one based on Open vSwitch or perhaps something like VPP.

*NOTE: Packet.net does support setting up VLANS. But I don't think you can do that through their API yet.*

### Architecture

Currently this deploys:

* 1 "controller" node that runs the kube-apiserver, kube-controller-manager and kube-scheduler
* 2 "worker" nodes that run kube-proxy, kubelet, containerd, fetch

With one controller it is not a highly available deployment. I have not tested three controllers yet, but it should work.

### Use of Ansible

I'm not (currently) using roles. Instead, because this is based on kthw, each step of the deployment is in its own standalone playbook, which are imported into `all.yml`. Not using roles was on purpose. In the future I may move to a role based model, but for now, it's a playbook based model.

### Security

The only major security consideration that has been made in this deployment is to try to ensure nothing but the k8s API is listening on the public IPs. Everything else should be listening on the private IPs only. Well, except whatever deployments you make public. :)

Otherwise no specific security considerations have been made.

### Why is this called Vent?

I didn't want to just call it "Kubernetes the Hard Way with Packet.net and Ansible" and I wanted to give it a name.

## Deploy Kubernetes

### Step 1: Provision Baremetal Servers

This example uses the [packet](https://github.com/ebsarr/packet) CLI. The easiest way to install the packet CLI is to have a go environment set up. But, one could create them in the Packet.net web interface as well. Nothing special is being done for provisioning at this time.

Here we will request some baremetal nodes from Packet.net.

*NOTE: We are using SPOT instances to keep costs down, but spot instances can go away at any time.*

Controller node:

```
export MAX_PRICE=0.30
export FACILITY=ams1
# Controller node
packet baremetal create-device --facility $FACILITY --spot-instance --spot-price-max $MAX_PRICE --plan c2.medium.x86 --hostname controller-0 -s --tags controller --os-type ubuntu_18_04
# Worker nodes
for s in 0 1; do
  packet baremetal create-device --facility $FACILITY --spot-instance --spot-price-max $MAX_PRICE --plan c2.medium.x86 --hostname worker-$s -s --tags worker --os-type ubuntu_18_04
done
```

Note that it can take 10+ minutes for the baremetal nodes to become available.

#### Why Not Use Ansible (or Terraform) to Provision the Servers?

Neither currently seems to support spot instances. I would expect that to change eventually though.

### Step 2: Setup a Place to Run Ansible From

Because I am in Canada and the Packet.net servers are in Amsterdam, I setup a small instance on Digital Ocean, in Amsterdam, from which to run Ansible. You might want to do the same just to speed up Ansible runs.

There is a requirements.txt file that can be used with Pip to setup a suitable environment.

I suggest using a Python virtual environment.

Eg.

```
virtualenv ~/ansible
source ~/ansible/bin/activate
pip install -r requirements.txt
```

### Step 3: Configure group_vars/all

There is an example `all` file to use.

```
cp group_vars/all.example group_vars/all
```

Create an encryption key.

```
head -c 32 /dev/urandom | base64
```

Example output:

```
gPsm8rqATXiHLxtfwe/pZK5a8LytlFeadH3hJXxeNqc=
```

Edit `group_vars/all` and add that key as the `ENCRYPTION_KEY` variable.

Example:

```
grep ENCRYPTION_KEY group_vars/all
ENCRYPTION_KEY = "gPsm8rqATXiHLxtfwe/pZK5a8LytlFeadH3hJXxeNqc="
```

### Step 4: Validate the Ansible Inventory Works

Once you have provisioned the servers and setup a place to run Ansible from, validate the Ansible inventory that comes with the project works.

The inventory script expects a `PACKET_API_TOKEN` environment variable.

```
# env | grep PACKET
PACKET_API_TOKEN=<your token>
```

It's possible to just run the inventory script by itself; it will just return JSON containing the inventory.

While the servers are provisioning, we would expect an empty inventory to be returned.

```
# ./inventory/packet_net.py
{
  "_meta": {
    "hostvars": {}
  }
}
```

Once the servers are active, we would expect to see something like the below.

```
# ./inventory/packet_net.py
{
  "someuuid": [
    "worker-1"
  ],
  "someuuid": [
    "worker-0"
  ],
  "_meta": {
    "hostvars": {
      "controller-0": {
        "ansible_host": "somepublicip",
        "packet_billing_cycle": "hourly",
        "packet_created_at": "2018-08-23T13:53:40Z",
        "packet_facility": "ams1",
        "packet_hostname": "controller-0",
        "packet_href": "/devices/someuuid",
        "packet_id": "someuuid",
        "packet_locked": false,
        "packet_operating_system": "ubuntu_18_04",
        "packet_plan": "c2.medium.x86",
        "packet_spot_instance": true,
        "packet_state": "active",
        "packet_tag_controller": "controller",
        "packet_termination_time": "",
        "packet_updated_at": "2018-08-23T14:00:49Z",
        "packet_user": "root"
      },
      "worker-0": {
        "ansible_host": "somepublicip",
        "packet_billing_cycle": "hourly",
        "packet_created_at": "2018-08-23T13:45:37Z",
        "packet_facility": "ams1",
        "packet_hostname": "worker-0",
        "packet_href": "/devices/someuuid",
        "packet_id": "someuuid",
        "packet_locked": false,
        "packet_operating_system": "ubuntu_18_04",
        "packet_plan": "c2.medium.x86",
        "packet_spot_instance": true,
        "packet_state": "active",
        "packet_tag_worker": "worker",
        "packet_termination_time": "",
        "packet_updated_at": "2018-08-23T13:52:49Z",
        "packet_user": "root"
      },
      "worker-1": {
        "ansible_host": "somepublicip",
        "packet_billing_cycle": "hourly",
        "packet_created_at": "2018-08-23T13:45:39Z",
        "packet_facility": "ams1",
        "packet_hostname": "worker-1",
        "packet_href": "/devices/someuuid",
        "packet_id": "someuuid",
        "packet_locked": false,
        "packet_operating_system": "ubuntu_18_04",
        "packet_plan": "c2.medium.x86",
        "packet_spot_instance": true,
        "packet_state": "active",
        "packet_tag_worker": "worker",
        "packet_termination_time": "",
        "packet_updated_at": "2018-08-23T13:53:03Z",
        "packet_user": "root"
      }
    }
  },
  "ams1": [
    "worker-1",
    "controller-0",
    "worker-0"
  ],
  "c2.medium.x86": [
    "worker-1",
    "controller-0",
    "worker-0"
  ],
  "someuuid": [
    "controller-0"
  ],
  "packet": [
    "worker-1",
    "controller-0",
    "worker-0"
  ],
  "tag_controller": [
    "controller-0"
  ],
  "tag_worker": [
    "worker-1",
    "worker-0"
  ],
  "testing": [
    "worker-1",
    "controller-0",
    "worker-0"
  ],
  "ubuntu_18_04": [
    "worker-1",
    "controller-0",
    "worker-0"
  ]
}
```

### Step 5: Validate Ansible Connectivity

If Ansible is setup properly, and if the SSH authentication is working, and we have IP connectivity, the following should be successful.

```
ansible -m ping all
```

Expected output:

```
controller-0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker-0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Step 6: Run the Deployment

All this is left is to run the entire deployment.

*NOTE: The deployment should only take 5 minutes or so, but if you have a poor connection it may be better to run the ansible command from a tux or screen session.*

```
ansible-playbook all.yml
```

Eg.

```
ansible-playbook all.yml
SNIP!
TASK [install kubedns deployment file] *******************************************************************************************************************************************************
changed: [controller-0]

TASK [deploy kubedns] ************************************************************************************************************************************************************************
changed: [controller-0]

PLAY RECAP ***********************************************************************************************************************************************************************************
controller-0               : ok=69   changed=46   unreachable=0    failed=0   
worker-0                   : ok=31   changed=20   unreachable=0    failed=0   
worker-1                   : ok=31   changed=20   unreachable=0    failed=0   
```

At this point you should have a functional Kubernetes deployment.

## Step 6: Configure Remote Access

For this step, you will need to run it from the node Ansible was deployed from. We are using files in the `fetched` directory. Otherwise, feel free to read the Certificate Authority and Kubernetes Configuration Files playbook to understand what is happening here.

On the Ansible host, install kubectl.

```
mkdir ~/bin/
wget https://storage.googleapis.com/kubernetes-release/release/v1.10.7/bin/linux/amd64/kubectl -O ~/bin/kubectl
chmod +x ~/bin/kubectl  
export PATH=$PATH:~/
```

Get the public IP of the controller.

```
ansible-playbook get-public-ip.yml
```

Example output:

```
PLAY [controller-0] **************************************************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************************************************
ok: [controller-0]

TASK [debug] *********************************************************************************************************************************************************************************
ok: [controller-0] => {
    "ansible_bond0.ipv4.address": "147.75.205.47"
}

PLAY RECAP ***********************************************************************************************************************************************************************************
controller-0               : ok=2    changed=0    unreachable=0    failed=0   
```

Now, to access the k8s cluster we need a kubeconfig file. We'll use pem files that were pre-created for a developer user.

*NOTE: Make sure to replace <your public ip> with the IP identified above.*

```
export K_USER=developer
export PUBLIC_IP="<replace with your public ip>"
mkdir ~/.kube
# if there is a kubeconfig already, move it to .bak
mv ~/.kube/config ~/.kube/config.bak
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=fetched/ca.pem \
  --embed-certs=true \
  --server=https://$PUBLIC_IP:6443 \
  --kubeconfig=$HOME/.kube/config

kubectl config set-credentials $K_USER \
  --client-certificate=fetched/$K_USER.pem \
  --client-key=fetched/$K_USER-key.pem \
  --embed-certs=true \
  --kubeconfig=$HOME/.kube/config

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=$K_USER \
  --kubeconfig=$HOME/.kube/config

kubectl config use-context default --kubeconfig=$HOME/.kube/config
cat $HOME/.kube/config
```

At this point there should be a config in `$HOME/.kube` that will be used to access the remote k8s deployment running in Packet.net.

## Step 7: Validate the Deployment

Now that k8s has been deployed, we should validate that it works.

Check nodes.

```
kubectl get nodes
```

Expected output:

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    1h        v1.10.7
worker-1   Ready     <none>    1h        v1.10.7
```

Alias kubectl to k just to do less typing.

```
alias k=kubectl
```

Get all.

```
k get all --all-namespaces
```

Expected output:

```
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE
kube-system   pod/kube-dns-fcf678fb4-xrpjd      3/3       Running   0          1h
kube-system   pod/kube-flannel-ds-amd64-8l57f   1/1       Running   0          1h
kube-system   pod/kube-flannel-ds-amd64-ktgz4   1/1       Running   0          1h

NAMESPACE     NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
default       service/kubernetes   ClusterIP   172.16.31.1    <none>        443/TCP         1h
kube-system   service/kube-dns     ClusterIP   172.16.31.10   <none>        53/UDP,53/TCP   1h

NAMESPACE     NAME                                     DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     2         2         2         2            2           beta.kubernetes.io/arch=amd64     1h
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0         0            0           beta.kubernetes.io/arch=arm       1h
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0         0            0           beta.kubernetes.io/arch=arm64     1h
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0         0            0           beta.kubernetes.io/arch=ppc64le   1h
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0         0            0           beta.kubernetes.io/arch=s390x     1h

NAMESPACE     NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/kube-dns   1         1         1            1           1h

NAMESPACE     NAME                                 DESIRED   CURRENT   READY     AGE
kube-system   replicaset.apps/kube-dns-fcf678fb4   1         1         1         1h
```

Run a simple shell.

```
kubectl create -f deployments/deployments-shell-demo.yml
```

Check if it's running.

```
# k get pods
NAME         READY     STATUS    RESTARTS   AGE
shell-demo   1/1       Running   0          19s
```

Get a shell on that pod.

```
# k exec -it shell-demo -- /bin/sh
/ # ip ad sh
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 0a:58:ac:10:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.3/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::90be:acff:fe23:7898/64 scope link
       valid_lft forever preferred_lft forever
/ # ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=122 time=0.717 ms
64 bytes from 8.8.8.8: seq=1 ttl=122 time=0.596 ms
64 bytes from 8.8.8.8: seq=2 ttl=122 time=0.660 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.596/0.657/0.717 ms
/ # exit
```

Note the IP address of `172.16.0.3/24` which is in the pod IP space.

We created a shell, exec-ed into it, and pinged Google's public DNS.

So far so good.

### Echoserver and NodePort

Create the echoserver.

```
kubectl run echoserver \
--image=gcr.io/google_containers/echoserver:1.4 \
--port=8080 \
--replicas=2
```

Expose its port.

```
kubectl expose deployment echoserver --type=NodePort
```

Now curl the NodePort on a public IP of one of the worker nodes.

```
curl <NODE_PUBLIC IP>:<NODE_PORT> -X POST -d "hello world"
```

Example output:

```
CLIENT VALUES:
client_address=172.16.0.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http:/<NODE_PUBLIC_IP>:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=<NODE_PUBLIC_IP>:<NODE_PORT>
user-agent=curl/7.47.0
BODY:
hello world
```
## TODO

* Run k8s processes non-root
* Consider running control plane and compute across all three nodes
* Flannel by itself doesn't support network policies -- https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/flannel
* Run sonoubouy against cluster
* Various k8s deployments are used but are continually run: flannel, rbac, kube-dns, etc. Need to check if they have already been deployed somehow.
* Use packet.net volumes for persistent volumes
* Additional bond0 route that should be removed
* 10.0.0.0/8 is used by packet, should not use 10.200.0.0/16
* Build custom ipxe image that will softraid root drives

## Issues

* FIXED: br_netfilter is required but is not loaded in Packet.net's default image
* componentstatuses - this doesn't affect the infrastructure

```
# kubectl get componentstatuses
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: getsockopt: connection refused   
controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: getsockopt: connection refused   
etcd-0               Healthy     {"health":"true"}     
```