## Deploying a sharded, production-ready MongoDB cluster with Ansible
------------------------------------------------------------------------------

- Requires Ansible 2.1+
- Expects CentOS/RHEL 7 hosts

### A Primer
---------------------------------------------

The above diagram shows how MongoDB differs from the traditional relational
database model. In an RDBMS, the data associated with 'user' is stored in a
table, and the records of users are stored in rows and columns. In MongoDB, the
'table' is replaced by a 'collection' and the individual 'records' are called
'documents'.  One thing to notice is that the data is stored as key/value pairs
in BJSON format.

Another thing to notice is that NoSQL-style databases have a looser consistency
model. As an example, the second document in the users collection has an
additional field of 'last name'.
 
### Data Replication
------------------------------------

Data backup is achieved in MongoDB via _replica sets_. As the figure above shows,
a single replication set consists of a replication master (active) and several
other replications slaves (passive). All the database operations like
add/delete/update happen on the replication master and the master replicates
the data to the slave nodes. _mongod_ is the process which is responsible for all
the database activities as well as replication processes. The minimum
recommended number of slave servers are 3.

### Sharding (Horizontal Scaling) .
------------------------------------------------

Sharding works by partitioning the data into separate chunks and allocating
different ranges of chunks to different shard servers. The figure above shows a
collection which has 90 documents which have been sharded across the three
servers: the first shard getting ranges from 1-29, and so on. When a client wants
to access a certain document, it contacts the query router (mongos process),
which in turn contacts the 'configuration node', a lightweight mongod
process) that keeps a record of which ranges of chunks are distributed across
which shards. 

Please do note that every shard server should be backed by a replica set, so
that when data is written/queried copies of the data are available. So in a
three-shard deployment we would require 3 replica sets and primaries of each
would act as the sharding server.

Here are the basic steps of how sharding works:

1) A new database is created, and collections are added.

2) New documents get updated when clients update, and all the new documents
goes into a single shard.

3) When the size of collection in a shard exceeds the 'chunk_size' the
collection is split and balanced across shards.


### Deploying MongoDB Ansible
--------------------------------------------

#### Deploy the Cluster
----------------------------
  
This deployment model focuses on deploying three shard servers,
each having a replica set, with the backup replica servers serving as the other two shard
primaries. The configuration servers are co-located with the shards. The _mongos_
servers are best deployed on separate servers. This is the minimum recommended
configuration for a production-grade MongoDB deployment. Please note that the
playbooks are capable of deploying N node clusters, not limited to three. Also,
all the processes are secured using keyfiles.

#### Prerequisite

Edit the group_vars/all file to reflect the below variables.

1) `iface: 'eth1'     # the interface to be used for all communication`.
		
2) Set a unique `mongod_port` variable in the inventory file for each MongoDB
server.

3) The default directory for storing data is `/data`, please do change it if
required. Make sure it has sufficient space: 10G is recommended.

### Deployment Example

The inventory file looks as follows:
		[all:vars]
		ansible_connection=ssh
		ansible_user=root # Change as per your requirement
		ansible_ssh_private_key_file= /path/to/your/key/file

		#The site wide list of mongodb servers
		[mongo_servers]
		mongo1 access_ip=<mongo1_ip> ansible_host=<mongo1_ip> ip=<mongo1_ip>
		mongo2 access_ip=<mongo2_ip> ansible_host=<mongo2_ip> ip=<mongo2_ip>
		mongo3 access_ip=<mongo3_ip> ansible_host=<mongo3_ip> ip=<mongo3_ip>

		#The list of servers where replication should happen, including the master server.
		[replication_servers]
		mongo3
		mongo1
		mongo2

		#The list of mongodb configuration servers, make sure it is 1 or 3
		[mongoc_servers]
		mongo1
		mongo2
		mongo3

		#The list of servers where mongos servers would run. 
		[mongos_servers]
		mongos1
		mongos2

Build the site with the following command:

		ansible-playbook -i hosts site.yml


#### Verifying the Deployment 
---------------------------------------------

Once configuration and deployment has completed we can check replication set
availability by connecting to individual primary replication set nodes, `mongo --host mongo1 --port 2700` 
and issue the command to query the status of
replication set, we should get a similar output.

		
		mongo1:PRIMARY> rs.status()
		{
			"set" : "mongo1",
			"date" : ISODate("2013-03-19T10:26:35Z"),
			"myState" : 1,
			"members" : [
			{
				"_id" : 0,
				"name" : "mongo1:2700",
				"health" : 1,
				"state" : 1,
				"stateStr" : "PRIMARY",
				"uptime" : 102,
				"optime" : Timestamp(1363688755000, 1),
				"optimeDate" : ISODate("2013-03-19T10:25:55Z"),
				"self" : true
			},
			{
				"_id" : 1,
				"name" : "mongo2:2700",
				"health" : 1,
				"state" : 2,
				"stateStr" : "SECONDARY",
				"uptime" : 40,
				"optime" : Timestamp(1363688755000, 1),
				"optimeDate" : ISODate("2013-03-19T10:25:55Z"),
				"lastHeartbeat" : ISODate("2013-03-19T10:26:33Z"),
				"pingMs" : 1
			}
			],
			"ok" : 1
		}


We can check the status of the shards as follows: connect to the mongos service
`mongo localhost:27017/admin -u admin -p 123456` and issue the following command to get
the status of the Shards:


		 
		mongos> sh.status()
		--- Sharding Status --- 
		  sharding version: { "_id" : 1, "version" : 3 }
		  shards:
			{  "_id" : "mongo1",  "host" : "mongo1/mongo1:2700,mongo2:2700,mongo3:2700" }
			{  "_id" : "mongo2",  "host" : "mongo2/mongo1:2701,mongo2:2701,mongo3:2701" }
			{  "_id" : "mongo3",  "host" : "mongo3/mongo1:2702,mongo2:2702,mongo3:2702" }
  		databases:
			{  "_id" : "test",  "partitioned" : true,  "primary" : "mongo1" }
