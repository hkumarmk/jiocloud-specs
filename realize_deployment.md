# intro

This is a plan for building out something that manages node

# data formats:

## for provisioning/updating stacks

in directory

environments/<environment>.yaml

roles:
  controller:
    cardinality: 2
    flavor: bare-metal
  compute:
    cardinality: 1
    flavor: bare-metal

This creates nodes called:

  <role><num>[\_<stack\_id>]

  NOTE: do we need stack id? If we always segment an etcd instance
  per test environment (and one for production)

# role assignment

Use Puppet regexp to map hostname (where we create hostnames
as a part of the provisioning process) to roles

node /<role>\d+/ {

}

# classification/datafication

A very simple userdata script is passed to all nodes when
they are initially provisioned. The script does just enough
to bootstrap Puppet.

## userdata script

* install Puppet
* pull down Puppet code and default data
* get hiera data from etcd (or somehow) (not sure if we need this here)
* write hostname to etcd
  * under the hosts that should be alive key (no ttl or crazy long ttl)
* run puppet
  * this Puppet run only sets up the cron job

## puppet from cron script

The cron job does the following:
  * check if our current version matches the desired version
  (this will be much more complicated for staging upgrades)
    if version is not installed
    * install data and puppet content for version
    * get hiera data from etcd
    * run Puppet
  * update the status of our node (did it do stuff, do nothing, or fail)
  * set current version in local file and etcd

# derive interconnectivity information

  This problem goes away, we can just query for the list of hosts
  from cron from etcd

  NOTE: At times we may need to execute certain code when the host list
  is updated, but this will be host specific.

# required changes

* update Puppet scripts to ensure that all role assignments are triggered
  from node declarations in site.pp
* ensure that all data that we need to override is set in hiera
* all roles have to be in Puppet (b/c userdata is going to be very simple)

# notes
