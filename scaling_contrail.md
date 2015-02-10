# Introduction

This document specified the plan for clustering contrail.

# Problem Statement

We need to cluster contrail is a way where nodes can arbitrarily leave and join
the cluster.

# Proposed Solution

This requires individual cluster management strategies for all of the nodes:

## rabbitmq

Clustering rabbitmq looks pretty straight forward. We will just have a seed node,
and have the other nodes join that node once it exists.

Based on the documentation and available configuration options for the
puppet rabbitmq module, clustering looks really easy.

* Create a shared secret for cluster members called the "erlang\_cookie"
* register all rabbit instances with consul
* have each instance query all consul to find other instances
* register each instance as a disc instance

### issues

* Right now I chose to do all as disks b/c it's easier (b/c one of them
has to be a disk instance). Rabbit docs recommend that at least one queue
in the cluster save to disk to ensure that the data won't be lost, otherwise,
they just run out of RAM.
* The way the module module works is to assume that when this changes, it should
completely wipe out it's local rabbitmq data. We need to be aware of this implication
if we ever decide to update the cookie.
* The nodes are currently not smart enough to tell Puppet to rerun when they don't have
the correct number of cluster members configured.

## Cassandra

In Cassandra, every cluster needs a seed to use as a starting place for establishing
the cluster's gossip protocol, Cassandra also recommends that not every node be the seed.

The seed node is provided as simple configuration to the Cassandra class.

This bootstrapping process anoints one of the Cassandra nodes as the seed. Non-seed nodes
block until seed.cassandra is registered in consul, and are passed a runtime\_fail resource
if the seed is not ready during compile (which should only happen on the initial run b/c of the
blocking resource). These two orchestration resources are used so that
each seed expects to fail exactly one time while initialing their Cassandra cluster.

I felt that the above describe pattern was pervasive enough to be expressed as a defined
resource type so that it can be applied to multiple use cases. The defined resource type:
`rjil::seed_orchestrator` was created for this purpose.

### issues

I initially tried to bootstrap Cassandra, making every node the seed and the results
were far from successful. Specially, it resulted in data loss during the initial
cluster bootstrapping which is a non-recoverable error.

If our seed node goes down, we may not be able to add new nodes to the cluster until it
comes back up. I don't think this is a problem, because the above mentioned orchestration
code should take care of this use case, but this scenario is something that needs to be
explicitly tested.

The fact that we saw data loss in our initial testing is troubling. I don't feel confident
to say that data loss is something that might not happen again.

## Zookeeper

Zookeeper requires both a seed node as well as a full list of every member on every member.
We implemented the seed node pattern in the exact same way as was done for Cassandra.

## contrail

I have not yet deployed contrail, my hope is that since we've already clustered it's stateful
backends, clustering contrail itself should be trivial.

We will probably have to ensure that the schema creation is only performed on the seed node.
