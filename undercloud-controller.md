# Undercloud controller

  It is an openstack controller with ironic driver for compute to provision
baremetal nodes. Jiocloud would use undercloud controller to provision all
baremetal nodes which host the overcloud (or the real, customer facing cloud)
openstack systems and other supporting servers. 

  It basically create a dhcp/pxe boot environment which can be managed using standard
openstack apis. This server also be served as dhcp, ntp servers for IPMI/remote
management interfaces, so it has a network interface connected to that network.
Using this setup, it manage auto-registration of baremetal nodes connected to
this network.
  
  This openstack controller is a single server all-in-one system (at this
moment) which contains all components that required.

Undercloud controller contain below components:

1. mysql database server
   This host the db for all openstack components for undercloud.

2. rabbitmq server
   rabbitmq is only used by ironic and neutron at this moment, others are using
zeromq. Earlier components need to be migrated to zeromq.

3. glance
    This hold both operating system image, kernel, ramdisk images, and
deployment kernel and ramdisk images. 

4. nova
   This is handling major part of provisioning baremetal systems as required. It
have ironic driver configured for compute - all nova components including all
nova controller components, ironic-api/conductor and nova compute are running on
same node.

   All nodes which provisioned by nova would always be doing pxe boot, and the
kernel and ramdisk is always provided by ironic.

  The improvements here may be to use IPA and other integrated tools with IPA
for more possible features.

5. ironic

Ironic have both ironic api which interact with nova-compute, which use ironic
conductor to deploy/provision the baremetal nodes.

ironic conductor use deployment kernel/ramdisk images to boot the nodes and
attach the server disk with iscsi (the ramdisk will make the node an iscsi
server, and share its first disk using iscsi) and then transfer the operating
system image to the node's disk.

5. keystone

6. neutron

  Here neutron implement a dhcp server using dnsmasq.

7. Dhcp, NTP server for IPMI or remote management interface network

  All servers' IPMI or out-of-band remote management interface would be
connected to a dedicated management network. Undercloud controller have extra 1G
connectivity to this management network and it run a dhcp server to provide IP
to those IPMI interfaces.

  The dhcp is also configured to call an auto-registration script which will
register the nodes automatically as it get an IP from this dhcp server. So in
effect, all servers which are connected to the management network (which have
dhcp enabled interface) will be automatically registered/added in ironic.

  It also running NTP server for the IPMI interfaces.

8. Scripts and ramdisks for baremetal management and node detail gathering

  There are couple of scripts and ramdisks for baremetal management like bios
setup, reordering pxe nics, gathering node details etc.

  These scripts need to be better and integrated in the deployment
workflow.

# Initial version

## Deployment code

   It is the intention to reuse as much code as possible from overcloud setup.
So undercloud controller setup is using almost same code as overcloud, except
few of them like contrail (undercloud use neutron with provider network), cinder,
object storage are not required in undercloud, ironic is only required in undercloud.

  Currently it is using overcast to provision it in virtual environment.

## Features

* Provision servers using IPMI and pxeboot
  This works with any kind of servers, no special hardware requirements are
required for this kind of setup. But here the problem would be that all nodes
will always do pxe booting and the barement systems kernel and ramdisk will be
provided by ironic.

  May be the improvement would be to use IPA - need more investigation to
understand the feasibility.

* Auto-registration of the nodes

  All nodes which have their IPMI interfaces connected to the management network
will get the IP from dhcp server which is running on same undercloud controller,
and they will get automatically regiestered on ironic, which will be available
to be used.

  There are some problems in current version, which would be getting fixed in
later versions:
  1. In case a server got removed/replaced, it will not get deregistered - there
is no process whether manual/automatic is defined.
  2. currently there are couple of scripts to be executed and in case of hp
nodes, an iso with custom ramdisk need to be booted to setup the system and to
gather node informations. This is not well integrated with deployment workflow
which need to be improved.

# Plan for future releases

* R&D on IPA or any other stuffs for better provisioning and easier
hardware/bios management.
* create a process to deregister the servers which are removed or replaced
* integrate the scripts and ramdisks with deployment workflow
* plan for the upgraded images or kernels
* avoid single point of failure for undercloud controller
