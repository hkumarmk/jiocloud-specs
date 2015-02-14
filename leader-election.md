# Introduction
  This document is indented to provide a leader election method for
various clustered or non-clustered setup/deployment without a dedicated leader
specified by system layout. Also the leadership should be transferred to a
different node if current leader has failed. The failure detection mechanism may
be different in different situations depending on the requirements.

There will be situations like the following which need a leader.

## Seed Node
  Here the node is the first node just for initializing the role, cluster or the
entire system and once the system is get initialized, the node does not have any special
meaning.

  Since the leader/seed node only relevant for initialization, there is no
failure detection mechanism and transfer of leadership required in this case.

## Cluster deployment
  In this case, one node (the leader) should initialize the cluster and others
should wait till certain point where they can join the leader to form a cluster.

## Restrict certain actions to be performed by single node in a role
  In this case, there may be certain actions to be performed by only a single
node within a role.

# Problem Statement

  Currently a dedicated leader is specified in the system layout who is sole
responsible for initializing the cluster, performing certain actions which is
only required to be perform only on single node within a role.

The major problem with the above approach is as follows:

1. This need a dedicated leader mentioned specifically in the layout and build
the deployment code to be executed around the hostname.
2. There is no leader transfer mechanism, so once the leader is down, it need to
be rebuilt to resume the funcionality done by the leader.
3. In case of seed node, it doesnt make sense to keep a leader role after the
initialization, which will never be used.
4. Also in certain scenarios, where the system initialization can cause wiping out or
corrupt existing system, keeping seed node role after first initialization is
going to be a problem as rebuilding the dedicated seed node can cause
reinitializing the system and thus corrupting the existing system/data.

# Proposed Solution

It is possible to develop a leader election system around consul.

## Consul session based leader election

### Concept
In this method, one session will be created per leader within the system. For
example, if there are two systems like ceph and contrail, each need leaders
within them, there will be two sessions created beforehand and the session will
be shared among all nodes within the group/role.

  The nodes who come up will try to acquire the lock by using acquire method while
creating a consul kv. This will return true or false, if true returned the lock
has been acquired by the local node and now he is the *leader*. If false is
returned, the lock has been acquired by some other node.

  Now the leader perform the actions what it need to do including initializing the
system/cluster and rest of the nodes may wait until the system is ready to
join the system or perform the actions itself.

  The session may be release gracefully by the leader itself, by operator or any
other logic in the system or automatically if leader healthcheck fails. In that
case other nodes may acquire the session and be act as leader. The level of
healthcheck and even the requirement of healthcheck is configurable, which will
be depends on the way the system use leader. It also accepts a timeout
(lock-delay) value upto 60 seconds. When a session invalidation takes place,
Consul prevents any of the previously held locks from being re-acquired for the
lock-delay interval which provide a safe guard on false positives.

  For example, in a case of mysql master/slave node, the healthcheck may need to be
good enough to decide if the mysql master is working or not and if failed, it
should release the lock which facilitates mysql slave to become master. Where in
case of seed node, no healthcheck is required at all as seed node doesnt have
any special meaning after first initialization.

NOTE: There are possibilities for false positives, which we may have to handle
either using lock-delay while creating session or a set of checks to validate
before transfer of leadership (or any other actions) or a combination of both.

### Implementation

  Each node participating the cluster will create a consul session by its
hostname. There should be a custom facter implementation per cluster service
which does the following.
* Create a consul session by the hostname of the node who participate the
cluster.
* Acquire the lock by using acquire=```<session id>``` parameter creating a
defined consul kv with value equal to its hostname. I propose to use the kv name
```services/<meta service>/<service>/leader```.
* Set facts
  ```
    <meta service>_<service>_leader with value of leader hostname
    orc_node_status with value 'leader' or 'follower' based on the status
  ```
  The fact orc_node_status can be used to build hiera hierarchy for leader and
followers.
* The facter should have a conditional to check the role from jiocloud_role
facter, so that it only run the facter code if the role need to join the
provided cluster/system.

It may also make sense to implement this logic in a function rather than a facter.

  Puppet code should use the facter/function to determine who is leader and
perform the actions accordingly. Also Puppet code may or may not perform
leadership transfer actions if current leader released the session.

NOTE: There are pros and cons on using facter and function for this purpose. The
advantage of facter is that it is available for entire resources where function
need to be called within the resources where it is required. Also facter can be
used in hiera to build hierarchy for leader. But the problem with facter is that
the session name should be pre-defined it cannot be configurable using hiera as
facter run happens before hiera.

#### Example

Here I assume an example of mysql replication (master/slave) cluster. Here I use
facter to acquire session.

* Define a session in hiera with name openstack-mysql-leader. May be something
  like the hiera data mentioned below
  ```rjil::consul::sessions
    openstack-mysql-leader:
      <param>: <value>
```
* Create a custom facter named openstack-mysql-leader which runs only on ocdb
  (or any other role defined) based on jiocloud_role facter and will try to
  acquire the session with name "openstack-mysql-leader" by creating a kv . If it succeed acquire
  session, it write a value of its hostname to the key openstack/mysql/leader.
  Also it return the key value, also it set the facter orc_node_status with value
  "leader". If it failed to acquire session, it just read the
  value of the above key and return the value which contain the leader node name
  and the facter orc_node_status="follower"
* Define a hierarchy in hiera based on the facter orc_node_status.
* Implemented rest of the puppet code around the facters to perform appropriate
  actions on leader as well as followers and to perform any leadership transfer
  in case of current leader released the session.


# Alternate Solutions and Pros/Cons

There are many alternate methods to achieve this some of them are mentioned
below.

## Consul CAS (check-and-set) KV based leader election
  Consul kv has a property called cas, which will have an index - this convert
PUT operations to a consul kv to check and set operations. If the index is 0,
then consul will only put the key if it does not already exist, so we can build
the leader election logic by one create a CAS based kv with index 0 and whoever
got to create the key will be the leader and the rest will be followers.

  The problem with this method is that the leadership transfer is going to have
a logic defined by external code as there is no healthcheck or anything there in
consul to automatically release the leadership.

  This mechanism may work well to build logic on rolling operations like rolling
upgrades where we can keep the index value to be "n" where n number of nodes in
a role can acquire the lock who can do the rolling operations like that, which
may be documented in different specs. It also may be used for internally
managing the node lifecycle.

## Statically define leader roles

The problem statement show the cons of this method.

## Using other tools like etcd, zookeeper to build a logic around

We use consul for service orchestration so it does not make sense to use
different toolsets.

# Considerations

Leadership should be the property of a node within roles[s] rather than a
statically defined role because of the issues mentioned iin the problem
description.

Also as much as logic possible to be performed destributed manner - each node
should be able to acquire the leadership independantly also they should be able
to decide its own status and who the leader is.

