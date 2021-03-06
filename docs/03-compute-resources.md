# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Network

In this section a virtual network will be setup to host the Kubernetes cluster.

Create the `kubernetes-the-hard-way` custom network using the `default` network as model:

Dump the default network configuration:
```
virsh --connect qemu:///system net-dumpxml default > default.xml
cp default.xml kubernetes-the-hard-way.xml
```
Edit the new kubernetes-the-hard-way.xml file
```
<network>
  <name>kubernetes-the-hard-way</name>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <ip address='192.168.10.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.10.2' end='192.168.10.254'/>
    </dhcp>
  </ip>
</network>
```
Create the network
```
virsh --connect qemu:///system net-define kubernetes-the-hard-way.xml
virsh --connect qemu:///system net-autostart kubernetes-the-hard-way
virsh --connect qemu:///system net-start kubernetes-the-hard-way
```

> The `192.168.10.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `kubernetes-the-hard-way` VPC network:

```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> output

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Debian](https://www.debian.org) Stretch, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

If you want to perform the complete installation over SSH, without X have a look at [this](https://github.com/telegrapher/debian-over-serial-port-howto)

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane.

Install the first VM, using virt-install
```
virt-install --os-type=Linux --os-variant=debian9 --name controller01 --ram=4096 --vcpus=1 --disk path=/opt/kubernetes/images/controller01.qcow2,bus=virtio,size=10 --cdrom /opt/kubernetes/debian*.iso --network network=kubernetes-the-hard-way --graphics none
```
It should bring automatically the console to perform the installation, but if we get disconnected for some reason, it can be reattached using:
```
virsh console controller01
```
After the installation, clone the two missing controllers from the first:
```
virt-clone --original controller01 --name controller02 --auto-clone
virt-clone --original controller01 --name controller03 --auto-clone
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
virt-install --os-type=Linux --os-variant=debian9 --name worker01 --ram=32768 --vcpus=8 --disk path=/opt/kubernetes/images/worker01.qcow2,bus=virtio,size=200 --cdrom /opt/kubernetes/debian*.iso --network network=kubernetes-the-hard-way --graphics none

virt-clone --original worker01 --name worker02 --auto-clone
virt-clone --original worker01 --name worker03 --auto-clone
```
## Common configuration

After cloning, we have 3 VMs using DHCP, now we need to give their proper configuration. In each one of them do:

- Install openssh-server
- Configure root access to the VM over SSH
- Configure the network to a static IP address instead of the default DHCP and with 192.168.10.1 as default gw.
  controller01 => 192.168.10.11/24
  controller02 => 192.168.10.12/24
  controller03 => 192.168.10.13/24
  worker01     => 192.168.10.21/24
  worker02     => 192.168.10.22/24
  worker03     => 192.168.10.23/24

Every VM should be autostarted:
```
for i in {01..03}; do virsh autostart controller${i}; done
for i in {01..03}; do virsh autostart worker${i}; done
```

### Verification

List the compute instances in your default compute zone:
```
virsh list
```
> output

```
 Id    Name                           State
----------------------------------------------------
 16    controller01                   running
 17    controller02                   running
 18    controller03                   running
 19    worker01                       running
 20    worker02                       running
 21    worker03                       running
```

Verify network configuration and SSH access:
```
for i in {1,2}; do for j in {1..3}; do ssh 192.168.10.$i$j uname -a ;done; done
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
