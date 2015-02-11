# Introduction
  The purpose of this document is describe the requirements and design
considerations for a horizontally scalable, redundant database system for
openstack. 

  The system should allow sharding the database tables to make it
scalable, and it should handle the data redundancy and failure scenarios
transparant to the application and without any administrative effort. This will
enable jiocloud system to achieve the following.

* Openstack system to scale thousands of nodes and hundreds of thousands of VMs
* Treat database servers like "cattles" where in case of failures/errors, they
   can be rebuilt without any serious problems.

# Problem Statement

Current mysql database solutions for openstack has two major disadvantages.

1. One of the solutions is Galera replicated database servers, in which case,
the data will be replicated all database servers and all servers are
read/wriable. The major problem with galera is its synchoronous nature of
replication cause large number of locks and even dead locks, which cause the
transactions fail during peak times. There are limitations on horizontal
scalability as all data replicated to all servers, and adding more servers cause
increasing locks and transaction failures.
2. Other existing solution is a binary file based replicated database which is
an asynchronous replication system. This will not cause any locks because of the
replication, but it will not ensure the data consistancy between master and
slave, so all reads have to be happen from master in case consistancy is
required. The scalability is another limitation of this solution.

# Proposed Solution

Vitess (https://github.com/youtube/vitess) - a set of servers and tools to
facilitate horizontal scalability of mysql databases. it does this by
transparantly sharding the mysql databases over multiple replicated mysql
servers. Also Vitess toolsets allows to transparantly failover backend mysql
servers in case of master failure.

Here is the logical diagram of vitess cluster components.

        SHARD 0                     SHARD 1               SHARD 2
     -------------               --------------       ----------------
     +-----------+               +-----------+         +------------+
     |   mysql   |               |   mysql   |         |   mysql    |
     | (master)  |               | (master)  |         |  (master)  |
     +-----------+               +-----------+         +------------+
     +-----------+               +-----------+         +------------+
     |  vttablet +---+     +-----+  vttablet |    +----+ vttablet   |
     +-----------+   |     |     +-----------+    |    +------------+
                     |     |                      |
     +-----------+   |     |     +-----------+    |    +------------+
     |   mysql   |   |     |     |    mysql  |    |    |   mysql    |
     |  (slave)  |   |     |     |   (Slave) |    |    |  (Slave)   |
     +-----------+   |     |     +-----------+    |    +------------+
     +-----------+   |     |     +-----------+    |    +------------+
     | vttablet  +---+     +-----+  vttablet |    +----+  vttablet  |
     +-----------+   |     |     +-----------+    |    +------------+
                     |     |                      |
     +-----------+   |     |     +-----------+    |    +------------+
     |   mysql   |   |     |     |  mysql    |    |    |   mysql    |
     |  (Slave)  |   |     |     | (Slave)   |    |    |  (Slave)   |
     +-----------+   |     |     +-----------+    |    +------------+
     +-----------+   |     |     +-----------+    |    +------------+
     |  vttablet +---+     +-----+  vttablet |    +----+ vttablet   |
     +-----------+   |     |     +-----------+    |    +------------+
                     |     |                      +---+
                     |     | +--------------------+   |
                     |     | |                        |
                   +----+----+++                   +--+------------+
             +-----+  vtgate   |                   | Topology data |
             |     +-----------+-------------------+  (Zookeeper,  |
             |     +-----------+-------------------+ etcd cluster) |
             +-----+  vtgate   |                   +------+--------+
             |     +-----------+                          |
             |                                            |
             |                                            |
        +----+----------------+                    +------+--------+
        |                     |                    |    vtctld     |
        |    Applications     |                    +---------------+
        |                     |
        +---------------------+

  Vitess consists of a number servers, command line utilities, and a consistent
metadata store. Taken together, they allow you to serve more database traffic,
and add features like sharding, which normally you would have to implement in
your application.

## Vitess Concepts

Here list major concepts of vitess system. Please refer
https://github.com/youtube/vitess/blob/master/doc/Concepts.md for
more details. Most of the concepts are just copied from this document.

### Cell
  A Cell is a group of servers and network infrastructure collocated in an area.
It is usually a full Data Center, or a subset of a full Data Center.
A Cell has an associated Topology Server, hosted in that Cell. Most information
about the tablets in a cell is hosted in that cells Topology Server. That way a
Cell can be taken down and rebuilt as a unit, for instance.

### Topology Server
  The Topology Server is the backend service used to store the Topology data,
and provide a locking service. Currently you can use Apache Zookeeper or etcd
and the plugins can be developed for other such applications like consul.

There is a global instance of that service. It contains data that doesnt change
often, and references other local instances. It may be replicated locally in
each Data Center as read-only copies.

There is one local instance of that service per Cell (Data Center). The goal is
to transparently support a Cell going down. When that happens, we assume the
client traffic is drained out of that Cell, and the system can survive using the
remaining Cells. 

### Keyspace
  A keyspace is a logical database. In its simplest form, it directly maps to a
MySQL database name. When you read data from a keyspace, it is as if you read
from a MySQL database. Vitess could fetch that data from a master or a replica
depending on the consistency requirements of the read.

When a database gets sharded, a keyspace maps to multiple MySQL databases, and
the necessary data is fetched from one of the shards. Reading from a keyspace
gives you the impression that the data is read from a single MySQL database.

### Shard
  A division within a Keyspace. All the instances inside a Shard have the same
data (or should have the same data, modulo some replication lag).

### Tablet

A tablet is a single server that runs:

* a MySQL instance
* a vttablet instance
* a local row cache instance
* an other per-db process that is necessary for operational purposes

It can be idle (not assigned to any keyspace), or assigned to a keyspace/shard.
If it becomes unhealthy, it is usually changed to scrap.

It has a type. The commonly used types are:

* master: for the mysql master, RW database.
* replica: for a mysql slave that serves read-only traffic, with guaranteed low
replication latency.
* rdonly: for a mysql slave that serves read-only traffic for backend processing
jobs (like map-reduce type jobs). It has no real guaranteed replication latency.
* spare: for a mysql slave not use at the moment (hot spare).

Only master, replica and rdonly are advertised in the Serving Graph.


## Vitess  components

### vtgate
  vtgate is a very light proxy that routes database traffic from your app to the
right vttablet, basing on the sharding scheme, latency required, and health of
the vttablets. This allows the client to be very simple, as all it needs to be
concerned about is finding the closest vtgate.

  vtgate listen on the port 15001, the status of vtgate can be accessed using
the url http://vtgate:15001/debug/status.

  Applications will connect to vitess using vtgate.

### vtctld
  vitess can be configured using vtctld also it can be used to explore topology.
It can be accessed using web interface using http://vtctld.example.com:15000.
The vtctl client can be used to access vtctld from commandline.

### Topology metadata store
  The topology is a metadata store that contains information about running
servers, the sharding scheme, and replication graph. It is backed by a
consistent data store, like Apache ZooKeeper. The topology backends are plugin
based, allowing you to write your own if ZooKeeper doesnt fit your needs. You
can explore the topology through vtctld.

### vttablet
  vttablet is a server that sits in front of a MySQL database, making it more
robust and available in the face of high traffic. Among other things, it adds a
connection pool, has a row based cache, and it rewrites SQL queries to be safer
and nicer to the underlying database.

  Usually vttablet and mysql server run on the same server. Vttablet use
memcache for raw query caching. So all mysql, memcache and vttablet would be
running on all backend database servers.

vttablet can be accessed using port 15100. The status can be accessed using the
url http://vttablet:15100/debug/status.

Here are the integral components that make vttablet working

#### mysql database
  vitess support mariadb version >= 10.x at this moment. Vttablet communicate
with database using mysql unix socket, which is faster than tcp/ip stack. Also
vttablet expect certain databases for its internal purpose:

* _vt database: This database have two tables which is used by vttablet for
    replications and other such purposes
* databases for keyspaces: There will be corresponding databases for each
   keyspaces.

#### vttablet application
  This application sits in front of mysql database and handle all database
traffic. 

#### row cache instance
  vttablet using memcache for row caching. Vttablet application start memcache
during its startup.

## Plan for vitess implementation and testing

The plan is to implement vitess in different phases, to reduce the impact. Here
is the phases

1. Setup single replicated set of vitess cluster without any sharding. Connect
openstack to the vitess, and test. This need development effort to patch
sqlalchemy to add a dialect for vitess or to add a similar interface from
openstack components. Only use mast node to read and write - usage of slave
nodes are only in case of master failure.
  1a. only use master for read and write
  1b. Start with zookeeper for topology datastore and later use etcd for that.
  1c. It may make sense to add/write plugin for consul for topology datastore.
2. Investigate on following items
  2a. Do we need to shard all tables? if not, What tables to be sharded across
      the openstack components.
  2b. Determine shard keys for all tables which are going to be sharded. There
      are lot of attention to be put here as cross-shard joins, queries can have
      performance issues (need to test the intensity) and cross-shard transactions are
      not acid-compliant.
3. Setup multi-sharded vitess database cluster and test openstack with this
  setup.
4. Determine the feasibility of considering eventual consistancy of database for
  openstack components, which makes us to use master for both read and write and
  to use slaves for reads. This gives us more scalability and load balancing
  between more nodes. Until this happen, we may try creating two vttablets
  on same servers and handlle in such a way that one node will have one master
  and one slave keyspace.
  
# Alternative Solutions (Pros/Cons)

## Mysql Sharding Solutions
There are number of mysql sharding solutions 
  * mysql fabric
  * spider storage engine
  * cubrid shard

Each of them have pros and cons, but none of them are proven to be as
scalable as vitess as of now.

#### Reference
* Cubrid - http://www.cubrid.org/manual/92/en/shard.html
* Spider storage engine - https://mariadb.com/kb/en/mariadb/spider-storage-engine-overview/
* Mysql Fabric - http://dev.mysql.com/downloads/fabric/

## Mysql ndb cluster
  MySQL Cluster is a technology that enables clustering of in-memory databases
in a shared-nothing system. The shared-nothing architecture enables the system
to work with very inexpensive hardware, and with a minimum of specific
requirements for hardware or software. MySQL Cluster integrates the standard
MySQL server with an in-memory clustered storage engine called NDB.

  Mysql cluster enables sharding and replication transparant to the applications
without needing changes in the drivers and applications. The applications can
access the mysql cluster using same mysql driver - SO NO DRIVER OR APPLICATION
CHANGES REQUIRED.

  NDB is disabled in mariadb - ndbcluster can be downloaded separately from
mysql upstream.
  
Mysql cluster have three components:

* Mysql database load balanced servers: 
  Here mysql database servers are stateless and can be considered to be run
behind load balancers.

  Mysql database servers connect to ndb data cluster through the network, so
both database servers and data node clusters can be scaled independantly. 

* NDB data node clusters
  This is a replicated, sharded nosql data structure stored in a cluster of data
nodes which transparantly managing replication, and sharding the data. It
support sharding the data to maximum 48 nodes. NDB is usually a memory based
storage engine; even though it support disk based tables with some restrictions.

  NDB cluster support handle online addition/deletion of servers, automatically
rebuild the data in case of node failure.

* NDB Cluster management daemon

  The management server is the process that reads the cluster configuration file
and distributes this information to all nodes in the cluster that request it. It
also maintains a log of cluster activities. Management clients can connect to
the management server and check the cluster status.


## Mysql cluster and vitess comparison

           Mysql Cluster               |               Vitess                
 --------------------------------------|-------------------------------------
  Synchronous replication              | Asynchronous Replication              
  Replication and Sharding Transparant to the application, no driver changes | Application/driver changes required, as vitess doesnt support mysql driver
  In-memory database with data persistance in disks | Normal innodb storage engine
  No changes required in the schema    | Each sharded table need extra column  
  All tables are sharded, no need to choose shard key, even though choosing right key will improve the performance  | Good amount of planning required on sharding to choose appropriate shard keys.
  Support arbitrary queries and txns across shards. | Cross-shard transactions are not acid compliant, may be performance problem on cross-shard queries.
  R/W load balancing with all nodes.   | Since it is asynchronous, read from laves can give you stale data.
  Scalability is limited to 48 nodes at this moment | Support theoritically unlimited number of nodes
  NDB is more complex, has more complex datastorage system, also data in ndb takes more space than innodb. | Use innodb storage engine which is better than ndb.
  NDB best suit for certain specific workloads and may have performance problems on certain workloads  | Innodb work well for most of the situations

# Considerations

  Considering very high scalability in mind, the database cluster must be horizontally
scalable as well as failure of the nodes should be handled transparantly and
gracefully.

# Unknowns

* At this moment, we dont have enough expertise in vitess.
* Openstack system rely on sqlalchemy to connect to a database which dont have a
 vitess driver. Also it is not tested on any problems on database queries made
 by openstack components work with vitess.
* All sharded tables need an extra column for shard key - this requirement can
 cause problems on openstack upgrades which contains any db updates as well as
 changing schemas can cause openstack to panic.
* Sharded keys to be chosen carefully as cross-shard queries will have
 performance problems and cros-shard transactions are not acid compliant. This
is going to be an ongoing activity as and when there is a schema change
* Vitess use asynchronous replication, so unless openstack system handle
  eventual consistancy in database data, only master can be used for reading.
  Currently openstack expect high consistancy database.
* Vitess is written in golang on which we dont have enough expertise.
* vtgatev3 is compliant to python dbapiv2, which is currently undergoing active
  development, currently vtgate v2 is active which is not compliant to python
  dbapi. 

