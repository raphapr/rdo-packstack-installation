![](http://latincloud.lccv.ufal.br/img/logos/lccv.png)

OpenStack
=========
Cloud computer systems are becoming used by many institutions as a way to keep availiable various kinds of services, like data storage, server hosting, among others. Keeping focus on IaaS ( Infrastructure as a Service), the Openstack stands as an important solution to offer scalability and service availability, supplying a highly flexible infra-structure with fast provisioning.

The OpenStack is a global collaboration of developers and technologists on cloud computing, resulting an ubiquitous and open-source computing platform for both public and private clouds. In practical terms, OpenStack is a set of softwares directed to the configuration and management of a cloud computing environment utilizing existing technologies on Linux environments.


Index
=========

* [1. Infrastructure and Network Settings](#1.-infrastructure-and-network-settings)
* [2. Preconfiguration Packstack](#2.-preconfiguration-packstack)
  * [2.2 Setting the hostnames](#2.2-setting-the-hostnames)
    * [2.2.2 Checking connectivity](#2.2.2-checking-connectivity)
  * [2.3 Setting the SSH](#2.3-setting-the-ssh)
* [3. Installing and executing Packstack](#3.-installing-and-executing-packstack)
  * [3.1 Repositories](#3.1-repositories)
  * [3.2 Executing the Packstack Installer](#3.2-executing-the-packstack-installer)
* [4. Post-Installation Configuration](#4.-post-installation-configuration)
  * [4.1 Setting the external network](#4.1-setting-the-external-network)
  * [4.2 Creating the external network](#4.2-creating-the-external-network)
  * [4.3 Creating the tenant network (tenant network)](#4.3-creating-the-tenant-network-(tenant-network))
* [5. Instancing a virtual machine](#5.-intancing-a-virtual-machine)
* [6. Notas importantes](#6.-notas-importantes)
* [Referências](#referências)

# 1. Infrastructure and Network Settings


Like was said previously, we work on GNU/Linux environment, specifically on CentOS and OpenStack version IceHouse. Thus this guide also works on GNU/Linux Fedora and Red Hat distributions.

We use three RUs SUN FIRE X4170m each one has:
* 2 processors Intel(R) Xeon(R) CPU X5570
* 24GB of RAM
* 128GB of disk
* 2 infiniband interfaces ( high performance interconnection )
* 4 Gigabit Ethernet interfaces
 
The instalation process was done on CentOS 6.5, using the instalation tool [packstack](https://wiki.openstack.org/wiki/Packstack). We consider a multi-node architecture with the Opestack Neutron node which requires the three kinds of nodes:

* **controller**: machine which hosts the managment services (Keystone, Glance, Nova, Horizon...)
* **network**: machine which hosts the network services and is responsible to supply the virtual web and to connect the virtual machines to the external web(neutron).
* **compute**: machine which hosts the virtual machines (hypervisor).


![](https://raw.githubusercontent.com/raphapr/rdo-packstack-installation/master/network.jpg)


In order to install OpenStack Multi-node you are going to need three network interfaces:

* **Management interface** (eth0): Network used to management, not accessible by the external network.
* **Interface for traffic between VMs** (eth1): Network used as internal network for traffic between virtual machines on OpenStack.
* **External interface** (eth2): This network is only connected to the network node in order to supply access to the virtual machines.

# 2. Preconfiguration Packstack

Before instaling OpenStack through packstack, we first prepared the network of the machines


## 2.1 Configurating the network 

### Controller node

On node controler, we configurated an interface for the managment (eth0) as follows:

Edit the file  "/etc/sysconfig/network-scripts/ifcfg-eth0" this way:

<pre>
# Internal Network
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.11
</pre>

### Network node

On network node, initially we set up two network interfaces.

/etc/sysconfig/network-scripts/ifcfg-eth0

<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.21
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1

<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.21
NETMASK=255.255.255.0
</pre>

### Compute node

On compute node, we set up two network interfaces.

/etc/sysconfig/network-scripts/ifcfg-eth0
<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.31
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1
<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.31
NETMASK=255.255.255.0
</pre>

## 2.2 Setting the hostnames

This stage must be done for every node.

After configurating the interfaces, restart the network service.
<pre>
# service network restart
</pre>
Set the machine name:

Edit the line which starts with ''HOSTNAME='' on file ''/etc/sysconfig/network'' as follows:
<pre>
HOSTNAME=controller
</pre>

Edit the file ''/etc/hosts'' as follows:
<pre>
127.0.0.1       localhost
192.168.0.11    controller
192.168.0.21    network
192.168.0.31    compute01
</pre>

### 2.2.2 Checking connectivity

From every node, ping a web site:

<pre>
# ping -c 4 openstack.org
PING openstack.org (174.143.194.225) 56(84) bytes of data.
64 bytes from 174.143.194.225: icmp_seq=1 ttl=54 time=18.3 ms
64 bytes from 174.143.194.225: icmp_seq=2 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=3 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=4 ttl=54 time=17.4 ms

--- openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3022ms
rtt min/avg/max/mdev = 17.489/17.715/18.346/0.364 ms
</pre>

Ping the network and compute from controller and vice versa.

**Warning: From this point, every configurations are made from controller.**

## 2.3 Setting the SSH

Set the SSH access without password through the RSA certificates of controller for the other nodes:

<pre>
# ssh-keygen -t rsa
# ssh-copy-id controller
# ssh-copy-id network
# ssh-copy-id compute01
</pre>

# 3. Instaling and executing Packstack

## 3.1 Repositories

Update the current packages on the system:
<pre>
# sudo yum update -y
</pre>

Configure the RDO repositories
<pre>
# sudo yum install -y https://rdo.fedorapeople.org/rdo-release.rpm
</pre>

Install the Packstack installer:
<pre>
# sudo yum install -y openstack-packstack
</pre>

## 3.2 Executing the packstack installer

Generate an _answers file_ with all the needed configurations for the installation of OpenStack environment:

<pre>
# packstack --gen-answer-file=packstack-answers-LCCV.txt
</pre>

For this installation, our file  _answer files_ differs from the default by these lines ahead:

<pre>
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_COMPUTE_HOSTS=192.168.82.31
CONFIG_NEUTRON_SERVER_HOST=192.168.82.21
CONFIG_NEUTRON_LBAAS_HOSTS=192.168.82.11
CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre
CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1000:3000
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1
CONFIG_NEUTRON_ML2_FLAT_NETWORKS=*
</pre>

+ [Complete answer file](https://github.com/raphapr/rdo-packstack-installation/blob/master/packstack-answers-LCCV.txt "Answers file")
+ [Table containing the informations of the configuration keys](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/3/html/Getting_Started_Guide/ch04s03s02.html "Table with info of the configuration keys")

Once the answers file is setted up, start the initialization through the command:

<pre>
# packstack --answer-file=packstack-answers-LCCV.txt
</pre>

# 4. Post-Installation configuration

## 4.1 Setting the external network


Assuming the external network is 192.168.84.0/24 and the network interface is **eth2**, edit the following configuration files on **network node**:

/etc/sysconfig/network-scripts/ifcfg-br-ex

<pre>
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.84.2    # your IP on the external network, may be any.
NETMASK=255.255.255.0  # your netmask
GATEWAY=192.168.84.1   # your gateway
DNS1=192.168.84.25     # your nameserver
ONBOOT=yes
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth2

<pre>
DEVICE=eth2
HWADDR=52:54:00:92:05:AE # MAC address
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
</pre>

Restart the network:

<pre>
# service network restart
</pre>

## 4.2 Creating the external network

On **controller node**, Source this file to read in the environment variables:

<pre>
# . /root/keystonerc_admin
</pre>

Inset the following commands:

<pre>
# neutron net-create ext-net --shared --router:external=True
</pre>

<pre>
# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.84.3,end=192.168.84.254 \
  --disable-dhcp --gateway 192.68.84.1 192.168.84.0/24
</pre>

## 4.3 Creating the tenant network (tenant network)

The tenant network is the internal network for VM access, its architecture isolates the network from others. Create it with the following commands:

<pre>
# neutron net-create demo-net 
# neutron net-create demo-net
# neutron subnet-create demo-net --name demo-subnet --dns-nameserver 8.8.8.8 --gateway 10.0.0.1 10.0.0.0/24
</pre>

**Create a router for the internal network and attach it to the external network and internal networks to it.**

<pre>
# neutron router-create demo-router
</pre>

Attach the router to the subnet created:

<pre>
# neutron router-interface-add demo-router demo-subnet
</pre>

Attach the router to the external network by setting it as a gateway:

<pre>
# neutron router-gateway-set demo-router ext-net
</pre>

Make sure the networks were created:

<pre>
# neutron net-list
# neutron subnet-list
</pre>

# 5. Instancing a virtual machine

Register on Nova your public RSA key:

<pre>
# nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key
</pre>

Register the CirrOS image of tests on the repositories of Glance:

<pre>
# glance image-create \
  --copy-from http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img \
  --is-public true \
  --container-format bare \
  --disk-format qcow2 \
  --name cirros
</pre>

Add rules to permit ICMP (ping) and SSH access:

<pre>
# neutron security-group-rule-create --protocol icmp default
# neutron security-group-rule-create --protocol tcp \
  --port-range-min 22 --port-range-max 22 default
</pre>

Finaly create the virtual machine:

<pre>
# nova boot --flavor m1.tiny --image cirros --nic net-id=8asc64-d8da-44b6-91cd-4acc87ee3500 --security-group default --key-name demo-key vm_teste
</pre>

See **net-id** of your network via **neutron net-list** command.

Free a floating IP address on the external network.

<pre>
# nova floating-ip-create external
+-------------+-------------+----------+----------+
| Ip          | Instance Id | Fixed Ip | Pool      |
+-------------+-------------+----------+----------+
| 192.168.84.3 | None        | None     | external |
+-------------+-------------+----------+----------+
</pre>

Assign a new instance:

<pre>
# nova add-floating-ip vm_teste 192.168.84.3
</pre>

Check the connectivity via ping/ssh:

<pre>
# ping 192.168.84.3
# ssh cirros@192.168.84.3
</pre>

Defautl CirroS password: cubswin:)

# 6. Important notes

## Is the internet connection too slow on the virtual machines?

On **Network node**, deactivate the *Generic Receive Offloaf* (GRO) on the external interface (eth2) through ethtool:

<pre>
ethtool -K eth2 gro off
</pre>

Add the command on the file /etc/rc.local to make the change to be kept after boot.

## Deleting a network on Neutron

In case you have issues when deleting a network on Neutron, the sequence of commands ahead may be useful:

<pre>
# neutron router-gateway-clear <router-id>
# neutron router-interface-delete <router-id> <tenant-network-id>
# neutron router-delete <router-id>
# neutron net-delete <tenant-network-id>
</pre>

# References

* http://oddbit.com/rdo-hangout-multinode-packstack-slides/
* http://docs.openstack.org/icehouse/install-guide/install/yum/content/
* http://www.monkey-code.com/blog/2013/10/04/building-a-multi-node-openstack-cloud/
* https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/
