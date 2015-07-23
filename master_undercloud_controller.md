# Introduction

  This document describes the spec for implementing a master undercloud
controller which is capable of easily provision and deploy undercloud
controllers in different environments.

# Problem Statement

  We are going to have multiple undercloud controllers or going to need build
undercloud controllers repeatedly in different situations:

1. There may be multiple test clusters which could include different undercloud
controllers
2. There are atleast three environments for pipeline where it need different
undercloud controllers - production, staging, prestaging
3. On scaling out production, we are going to need multiple subnets which need
multiple undercloud controllers for production.
4. CICD of undercloud controller - currently gate and at of undercloud
controller is happening in virtual environments, but undercloud controller
cannot be completely tested on virtual environments, so it is necessary to do
acceptance testing on baremetal environment (say pre-staging). In order to have
this happen, it need undercloud controller to be built on every test.

Currently provisioning undercloud conroller involves manual tasks like to boot
operating system on it, setup puppet in that system etc, which need to be
solved.

# Proposed Solution

  Setup a master undercloud controller (muc) which would be yet another openstack with
ironic driver with some difference from regular undercloud controller. The
difference will be mostly on network side because in case of muc, there are
going to have multiple subnets - each undercloud controller(uc) which are spawned by
muc will be in different vlans and different subnets. So there are going to have
two major changes in this sense:

1. Changes regarding to neutron/network

  This would need some experiments.

  1.1. No neutron, use isc-dhcpd - The dhcp service on muc should support
multiple networks - it seems dnsmasq would not be able to support this, so may
be isc-dhcpd need   to be used here, in which cause ironic will not be using
neutron, but would be using isc-dhcpd.

  1.2. Try using multiple flat-networks in neutron, in this case, neutron will
setup different dnsmasq daemons for each network, I think this may need
different physical interface (or may be an alias or something) per network, need
to do some investigation on this

  1.3. DHCP will provide static mac-binded ip address to both data interface and
ilo interfaces for uc nodes

  1.4. muc dhcp will only provide IP addresses based on mac addresses, no
dynamic ip addresses will be provided. This is important because network devices
will be configured with dhcp-relay which relay dhcp traffic from all
vlan/subnets (where any undercloud controller exist), to this node.

  1.5. This implementation will add one more dhcp server on all vlans/subnets
along with the dhcp server which added by undercloud controller. So there will
be two dhcp servers on any subnet for undercloud -
    a. A local dhcp server (one on data network and one on ilo) served by
undercloud controller which provide ip details to all baremetal/ilo systems but
the undercloud controller itself. This dhcp server will only be exposed to local
network/subnet/vlan.

    b. dhcp server (again one on data network and one on ilo network) from
master undercloud controller - network devices will expose that dhcp server with
dhcp-relay. This will not answer dhcp requests for the mac addresses for any
baremetal servers but undercloud controller mac address which have registered in
it.

2. Changes regarding network device setup
  Network switches to implement dhcp-relay which expose muc's ip address[s] to
all required vlans.

# Alternative Solutions

May be cobbler or something, but it is going to be entirely different solution
from what is used otherwise.

# Considerations

* There will be some complexity involved on having two different dhcp server
configured on a network and setting up dhcp-relay on all vlans.

# Unknowns

