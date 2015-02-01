# Hadoop Admin Cert

# Meet Hadoop:

realtional data is often normalised to retain integrity and remove redundancy. Normalization poses problems for mapreduce because it makes reading a record a nonlocal operation. one of the central assumptions that MapReduce makes is that it is possible to perform high speed streaming reads and writes.


# HDFS (17%)

### Design of HDFS:

There are a number of specific goals for HDFS:

• Store millions of large files, each greater than tens of gigabytes, and filesystem sizes
reaching tens of petabytes.

• Use a scale-out model based on inexpensive commodity servers with internal JBOD
(“Just a bunch of disks”) rather than RAID to achieve large-scale storage. Accom-
plish availability and high throughput through application-level replication of data.

• Optimize for large, streaming reads and writes rather than low-latency access to
many small files. Batch performance is more important than interactive response
times.
7
• Gracefully deal with component failures of machines and disks.
• Support the functionality and scale requirements of MapReduce processing

* very large files

* Streaming data access
	HDFS built around idea: most efficient data processing pattern is write once, read many times pattern

* Commodity hardware:
	doesnt require expensive hardware, designed to run commodity hardware. fault tollerance.

* low-latency data access:
	applications with low-latency access to data will not work well with hdfs. hdfs is optimized for delivering a high throughput of data. hbase is better choice for low-latency.

* lots of small files:
	namenode holds filesystem metadata in memory, the limit to the number of files in a fs is governed by the amount of memory on the nn. each file, directory and block take up 150 bytes. e.g. one million files = 300mb in metadata in mem.

* Multiple writes, arbitary file modifications:


### HDFS conecepts:

* BLOCKS

	a disk had a block size, wich is the min amount of data that it can read or write. Filesystems blocks = few kb, disk blocks = 512 b. filesystem matience include `df` and `fsck` - operate on a block level.

	hdfs blocks are much larger - 64 MB by default.  files in hdfs are broken into block sized chunks which are then stored as independent units.

	the hdfs blocks are large to minimise the cost of seeks.

	benefits:
havin block abstraction for a distributed fs means a file can be larger than any single disk in the network.

	simplifies the storage subsystem.. as blocks are fixed sizes it is easier to calculate ho many can be stored on a disk

	each disk is replicated to a small number of physically seperate machines (default = 3) --> client will not notice if a replicated block is corrupted. part of data integrity model (page 81)

	``` hadoop fsuk / -files -blocks``` will list the blocks that make up each file in fs.

* NAMENODES AND DATANODES:

	the namenode manages the filesystem namespace. it maintains the fs tree and the metadata for all the files and directories in the tree. all stored in the namespace image and the edit log. it also knows the datanodes on which all the blocks for a given file are located. does not store block location.

	Datanodes are the workhorses of the filesystem. they store and retrieve blocks when the are told to. they give reports of blocks they are storing.

	it is important to make the nn resilient to failure. hadoop does this by:

	1) buck up files that make up the persistent state of the fs metadata.

	2) it is possible to run a secondary namenode (does not act as a nn) its main role is to periodically merge the namespace image with the edit log to prevent the edit log from becoming too large.

* HDFS FEDERATION:

	hdfs fed allows a cluster to scale by adding namenodes, each of which manages a portion of the fs namespace. e.g one nn might manage all files rooted under /user, and second manages /share.

	to access a fed hdfs cluster, clients use client side mount tables to mpa file paths to namenodes. config using ViewFileSystem

	The one major way in which namenode federation is different from running several discreet clusters is that each datanode stores blocks for multiple namenodes. More precisely, each datanode has a block pool for each namespace. While blocks from different pools are stored on the same disks (there is no physical separation), they are logically exclusive. Each datanode sends heartbeats and block reports to each name-node.

* HDFS HIGH-AVAILABILITY:

	the namenode is still a single point of failure even with a fed. (SPOF). the namenode is the sole repo of the metadata and the file-to-block mapping. if the nn fails, everything fails.

	to recover:
		admin starts a new primary nn with one of the fs metada replicas and configures datanodes and clients to use this nn. needs to:
			1) load its namespace image into memory
			2) replaces its edit log
			3) recieved enough block reports from the datanodes to leave safe mode.

	spoort for HDFS High availability (HA):
		there is a pair of namenodes in an active standby config. event of failure of active nn, the standby takes over its duties to continue servicing client requests.

		requirements:
			1) must use shared storage to share edit logs, requires NFS filer
			2) datanodes must send reports to all nn instances.
			3) clients must be configed to handle nn failover

* NAMENODE HIGH AVAILABILITY

	deployed as a pair of namenodes. the edits write ahead log needs to be available on both nn so stored on shared storage (NFS)/ as the active namenode writes to the edits log, the standby namenode is constantly replaying transactions to ensure it is up to date and ready to take over in case of failure.

	two types of failover:
*	graceful failover - initiated by admin
*	nongraceful failover - the result of a detected fault in the active process




### The Command-Line Interface:

fs.default.name e.g. hdfs://localhost/   	: used to set default fs for hadoop
hdsf port  8020

dfs.replication 			: to config replication rate e.g 1

* copy from local
	```hadoop fs -copyFromLocal local/input/text.txt hdfs://localhost/user/bob/text.txt```
	-copyToLocal


### Network Topology and hadoop
	 in the context of high-volum data processing, 'closness' is the rate at which we can transfer data between nodes. bandwidith between nodes can be used to measure distance


### Data Ingest with Flume and sqoop

Apache flume is a system for moving large quantites of streaming data into hdfs. flume nodes can be arranged in arbitary topologies. flume offers different levels of delivery reliability from best effort delivery, to end-to-end - garuntees delivery even in the event of multiple flume node failures.

apache sqoop designed for performing bulk imports of data into hdfs from structured data.

(parallel copying with distcp)
it is also possible to act on a collection of files - by specifying file globs. distcp allows for copying large amounts of data to and from hadoop filesystems in parallel. e.g. for transfering data between two hdfs clusters
```hadoop distcp hdfs://namenode1/foo hdfs://namenode2/bar```

the distcp job is a mapreduce in of itself. (literally just a map job that simply copies). each map typically copyings around 256mb each. e.g. copying 1,000 GB of files to a 100-node cluster will allocate 2000 maps (20 per node) so each will copy 512 mb on average. the arg can be specified using -m e.g. ```-m 1000``` would allocate 1000 maps.

requirement:

	clusters must run identicial versions of hadoop as the RPC systems are incompatible. however the job can still be done by using read-only HTTP based HFTP filesystem to read from source e.g

```hadoop distcp webhdfs://namenode1:50070/foo webhdfs://namenode2:50070/bar```

### Hadoop Archives

hadop stroes small files inefficiently, since each file is stored in a block, and block metadata is held in memory by the namenode, therefore a large number of small files eat up a lot of memory on the nn. (these small files do not take up a block's size worth of data on disk)

Hadoop Archives (HAR) files packs files into hdfs clocks more efficiently therefore reducing nn mem usage.

the archive tool runs a mapreduce job to process input files in parallel.
e.g. ``` hadoop archive -archiveName files.har /my/file /my```

	Limitations:

		* creating a har creates a copy of the original files, so you need as much disk space as the files you are archiving to create the archive - can delete original after.

		* har files are immutable once they have been created. to add or remove files you must re-create



#  2.0 YARN and MapReduce version 2 (MRv2) (17%)

the framework that is used for execution is set by the ```mapreduce.framework.name``` property which takes the values 'local', 'classic' (MR1) and 'yarn' (new framework)

yarn was created due to the fact that for very large clusters in the region of 4000 nodes +, the classical MR system begins to hit scalability botlenecks

YARN remedies the scability problembs of the previous version by splitting the responsibilites of the jobtracker into seperate entities.
	the jobtracker takes care of both scheduling (matching tasks with tasktrackers) and task progress monitoring (keeping track of tasks, restarting failed or slow tasks, task bookkeeping).

YARN seperates these two roles into two independent daemons : ```resource manager``` to manage the use of resources across the cluster. and an ```application master``` to manage the lifecycle of applications running on the cluster. the application master negotiates with the resource manager for cluster resources

these resources are described as containers with a fixed amount of memory (applications are run in these containers). containers are overseen by the ```node manager```

### Entities involved in YARN:

* the client, which submits the MR job
* the YARN resource manager, ehich coordinates the allocation of compute resources on the cluster
* the YARN node manager, which launch and monitor the compute containers on the machines in the cluster.
* the MR application master which coordinates the tasks running the MR job. the application master and the MR tasks run in containers that are scheduled by the resource manager and managed by the node managers.
* the HDFS which is used for sharing job files between the other entities.

```note``` a small MR task is considerated to be small when the  input size is less than one hdfs block and the job is assigned less than 10 mappers.

``` get figure  6-4 page 198```


#### Job submission
##### step 1
	* has implementation of ```ClientProtocol`` that is activated when ```mapreduce.framework.name``` is set to yarn
	* submission process similiar to classic MR
	* the new job ID is retrieved from the ```resource manager```

##### step 2
	* the job client checks the output specification of the job
	* computes input splits
	* copies job resources (inc the job jar, config, and spit information) to HDFS

##### step 3
	* the job is submitted by calling ```submitApplication()``` on the resource manager

#### Job Initialization
##### step 4
	* the resource manager hands off the request to the scheduler.
	* the scheduler allocates a container, and the resource manager then launches the application master's process there, under the node manager's management

##### step 5a 5b
	* the application master for the MR jobs is a java application whose main class is MRAppMaster.
	* it initializes the job by creating a number of bookkeeping objects to keep track of the jobs progress

##### step 6
	* retreieves the input splits from hdfs.

##### step 7
	* it then creates a map task object for each split, as well as a number of resuce task objects.
	* next the application master decides hoe to run the tasks that make up the MR job

#### Task assignment
	* the application master requests containers for all the map and reduce tasks in the job from the resource manager

##### step 8
	* the scheudler uses the split metadata to make scheduling decisions. it attemps to place tasks on data-local nodes, if not possible then rack local.
	* both map and reduce tasks are allocated 1 GB of memory. this can be configured by changing ```mapreduce.map.memory.mb``` and ```mapreduce.reduce.mb```
	* in YARN, applications may request a memory capability that is anywhere between the min and max allocation, must be a multiple of the min allocation. the default mem allocation is 1024 MB *set by ```yarn.scheduler.capacity.minimum-allocation-mb```. default max is 10240 MB set by ``yarn.scheduler.capacity.maximum-allocation-mb```. tasks can request anaywhere between this.

#### Task execution
##### steps 9a 9b
	* once a task has been assigned a container by the resource manager's scheduler, the application master starts the container by contacting the node manager.
	* the task is executed by a java application ehose main class is ```YarnChild```. before it can run, the task it localizes the resources that the task needs, inclusing the job config and the JAR file and any files in the distributed cache

##### step 10
	* it runs the map or reduce tasks

##### step 11
	the YarnChild runs in a dedicated JVM, to isolate the user code from the long-running system daemons. yarn does not support JVM reuse, so each task runs in a new JVM.

#### Job completion

	as well as polling the application master for progress, every 5 seconds the client checks whether the job has the completed by calling the ```waitForCompleteion()``` method on Job.

### Faliures in YARN

Things that can fail:
* the task
* the application master
* the node manager
* the resource manager

##### Task Failure
* runtime exceptions and sudden exits of the JVM and hanging tasks are propgated back to the application master.
* a task is marked as failed after four attempts. ```mapreduce.map.maxattempts```

##### Application master failure
* applications in YARN are tried multiple times in the event of failure. default applications are marked as failued if they fail once.
* ```yarn.resourcemanager.am.max-retries```
*an application master sends periodic heartbeats to the resource manager, if the app master fails, the resource manager will detect the failure and start a new instance of the master running in a new container.
* ```yarn.app.mapreduce.am.job.recovery.enable ``` is a  boolean

#### Node manager failure
* if a node manager fails it will stop sending heartbeats to the resource manager, and the node manager will be removed from the resource manager's pool of available nodes.
* default: 10 minutes, determines the time the resource manager waits before condifering a node manager that has sent no heartbeat during that period as failed.
* any task or application master running on a failed node manager will be recovered.
* node managers may be blacklisted if the number of failures for the application is high.

#### resource manager failure
* if the rm fails, neither jobs nor task containers can be launched.
* after a crash, a new resource manager instance is brought up (by admin) and it recovers from saved state
* the storage used by the resource manager is configed via ```yarn.resourcemanager.store.class``` property


### Job Scheduling

previous versions of hadoop scheduled jobs in order of submission, called the FIFO scheduler. each job would use the whole cluster. not good for shared cluster usage.

#### The Fair Scheduler

aims to give every user a fair share of the cluster capacity over time.
if a single job is running, it gets all of the cluster
jobs are placed in pools, by default each user gets their own pool
supports preemtion, so if a pool has not recieved its fair share for a certain period of time, the scheduler will kill tasks in pools running over capacity in order to give more slots to the pool running under capacity
`org.apache.hadoop.mapred.FairScheduler`


#### The Capacity Scheduler

* A cluster is made up of a number of queues (like pools), which may be hierarchical (parent/child of another queue), each queue has an allocated capacity.
* within each queue, jobs are scheduled using FIFO scheduling (with priorities)


### MapReduce Configuration Tuning

* general principle is to give the shuffle as much memory as possible. however this does have a tradeoff, you need to make sure that the map and reduce functions get enough memory to operate.
* amount of mem given to JVMS in which the map and reduce tasks run is set by ```mapred.child.java.opts``` property. make this as large as posible
* on the map side, best performance obtained by avoiding multiple spills to disk, one is optimal. ```io.sort.*``` min the number of spills. increase ```io.sort.mb```
* on the reduce side, best performance obtained when the intermediate data can reside entirely in memeory. if the reduce side has light mem requirements, setting ```mapred.inmem.merge.threshold``` to 0 and ```mapred.job.reduce.input.buffer.percent``` to 1.0 is optimal.

note: table 6-1 page 213




# YARN (def guide 4th eddition)

yarn provides its core services via two types of long-running daemons:
* a Resource Manager (one per cluster)
* a Node Manager : runs on all the nodes in the cluster to launch and monitor containers.

a container executes an application-specific process with a constrained set of resources (memory, CPU etc).

to run an application, a cleint contacts the RM and asks it to run an application master process. the RM then finds a node manager that can launch the application master in a container. the application could then request more containers from the RM to use them to run a distributed computation.

#### Resource Requests

flexible model.locality is critical in ensuring that the distributed data processing algorithms use the cluster bandwidth efficiently, so yarn allows an application to specify locality constraints for the containers it is requesting. e.g. specific node, rack etc

if locality constraint cannot be met, either no allocation is made, or the constraint can be loosened. e.g. start a container on the same rack that the application requestion the node on.

common task of launching a container to process a HDFS block, the application would request a container on one of the nodes hosting the block's three replicas, or on a node in one of the racks hosting the replicas, or failing that, on any node in the cluster.

#### Application lifespan

categorised in terms of how they map to the jobs that users run.

1) one application per user job, which is the appraich in that MR takes.
2) second mondel is to run one application per workflow or user session of jobs. containers can then be reused between jobs, potential to share cache intermediate data.
3) long running application that is shared by different users. coordination role. used by impala. to provide a proxy application that the impala daemons communicate with to request cluster resources. the "always on" application master means that users have very low-latency responses to their queries since overhead of starting a new application master is avoided.

In yarn, the resource maanger and an application master (one for each MR job). the jobtracker is also responsible for storing job history for completed jobs, although it is possible to run a job history server as a seperate daemon to take the load off the jobtracker. in yarn the equivalent role is the timeline server, which stores application history

#### comparison between MR1 and YARN

MapReduce 1 		YARN
Jobtracker 			Resource manager, application master, timeline server

Tasktracker 		Node manager

Slot 				Container

#### limitations of MR1

hadoop operations page 85

In YARN, a node manager manages a pool of resources, rather than a fixed number
of designated slots. MapReduce running on YARN will not hit the situation where
a reduce task has to wait because only map slots are available on the cluster, which
is the case in MapReduce 1. If the resources to run the task are available, then the
application will be eligible for them

n some ways, the biggest benefit of YARN is that it opens up Hadoop to other types
of distributed application beyond MapReduce. MapReduce is just one YARN ap‐
plication amongst many.

#### Scheduling in YARN

in a busy cluster, an application will often have to wait to have some of its requests fulfulled. tis the job of yarn scheduler to allocate resources to applications according to some defined policy

##### The FIFO scheduler
places applications in a queue and runs them in the order of the submission. requests for the first application in the queue are allocated first, when finished, the next app in the queue is served.

does not need any config. not sutiable for shared clusters. in a shared cluster it is better to use the Capacity scheduler or Fair scheduler

With the Fair Scheduler (iii. in Figure 4-3) there is no need to reserve a set amount of
capacity since it will dynamically balance resources between all running jobs.

#### The Capacity Scheduler

allows sharing of a hadoop cluster in organisations. each organisation is allocated a certain capacity of the overall cluster. each organisation is set up with a dedicated queue that is configs to use a given fraction of the cluster capacity. queues may be further divided in hierarchical fashion.

a single job does not use more resources than its queues capacity. however if there is more than one job in the queue and there are idle resources available, then the Capacity scheduler may allocate the spare resources to the jobs in the queue even if that casues the queues capacity to be exceeded (queue elasticity)

capacity-scheduler.xml

If the property yarn.scheduler.capacity.<queue-path>.user-limit-factor is set to a value larger
than 1 (the default), then a single job is allowed to use more than its queue’s capacity.

#### The Fair Scheduler
The Fair Scheduler attempts to allocate resources so that all running applications get
the same share of resources.

Enabling the Fair Scheduler
The scheduler in use is determined by the setting of yarn.resourcemanager.schedu
ler.class . The Capacity Scheduler is used by default (although the Fair Scheduler is
the default in some Hadoop distributions, such as CDH), but this can be changed by
setting yarn.resourcemanager.scheduler.class in yarn-site.xml to the fully-
qualified classname of the scheduler, org.apache.hadoop.yarn.server.resourceman
ager.scheduler.fair.FairScheduler .

onfigured using an allocation file named fair-scheduler.xml that
is loaded from the classpath. (The name can be changed by setting the property
yarn.scheduler.fair.allocation.file .)



# Setting up a hadoop cluster

### Cluster specifications
	in 2010, the standard:
	processor: two quad-core 2-2.5 GHz CPUs
	Memory: 16-24 GB ECC RAM
	Storage: four 1 TB SATA disks
	network: Gigabit Ethernet

#### why not RAID (Redundant Array of Independent Disks)
	HDFS cluster does not benefit from using RAID for datanode stoage. the redundancy that RAID provides is not needed, since HDFS handles it by replication between nodes.

	RAID 0 read and write operations are limited by the speed of the slowest disk in the RAID array. rather use JBOD (just a Bunch Of Disks)

	RAID failure of a single disk causes the whole array to become unavailable (therefore the whole node)

### how big?

	e.g. if your data grows by 1 TB a week and you have three-way HDFS replication, you need an additional 3 TB of raw storage per week. allow 30% for intermediate files and logfiles... therefore one extra machine with the above specs each week.

	therefore a cluster that holds two years of data needs 100 machines.


### Network Topology

	typically there are 30-40 servers per rack, with a 1 GB switch for the rack and an uplink to a core switch or router.

	the aggreagate bandwidth between nodes on the same rack is much greater than that between nodes on different racks

	#### Rack awareness
	* best performance to configure hadoop so it knows the topology of the network. if single rack no more work needed (default).
	* for multirack clusters, you need to map nodes to racks. therefore hadoop will prefer within-rack transfers to off-rack transfers
	* the Hadoop config must specify a map between node addresses and network locations. ```DNSToSwitchMapping```
	* ```topology.node.switch.mapping.imp1 defines implementation. the namenode and the jobtracker use to resolve worker node network locations

### Cluster Setup and installation

#### install java
	java 6 or later, latest stable SDK

#### create hadoop user
	* good practice to have dediccated hadoop user account
	* small clusters: user's home dir an NFS-mounted drive, to aid with SSH key distribution.
	* autofs allows you to mount the NFS filesystem on demand when system access it.

#### installing hadoop
	* after download to /usr/local
	sudo tar
	sudo chown -R hadoop:hadoop hadoop-install....

#### ssh config
	hadoop control scripts reply on ssh to perform cluster-wide operations.
	ssh needs to be set up to allow password-less login for the hadoop user from machines in the cluster. to achieve this. generate a public/private key pair and place it in an NFS location that is shared across the cluster.
``` ssh-keygen -t rsa -f ~/.ssh/id_rsa```
	*use an ssh-agent to avoid the need to enter a password for each connection
	* ensure that the public key in in the ~/.ssh/authorized_keys file on all the machines in the cluster that we want to connect to.
	* if the hadoop user's home dir is an NFS filesystem, the keys can be shared across the cluster:
			```cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys```


### Hadoop Configuration

#### Configuration management
there is no single/ global location for config info. each hadoop node in the cluster has its own config files.
* there are tools for synchronizing config using ```rsync```. or sdh or pdsh
* tools such as Chef, Puppet, cfengine and bcfg2 are used to maintain classes of cluster management with predefined config.

#### control scripts
* there are many scrips in the bin dir for starting/stopping daemons etc across the whole cluster. first you need to tell hadoop which machines are in the cluster.
* enter: `master`(i.e. machine that will determine secondary namenode) and `slaves` (i.e. datanodes and tasktrackers, each contains a list of machine host-names or IP addresses. all in hadoop-env.sh
* start-dfs.sh : starts all the hdfs daemons in the cluster

#### master node scenarios
* on a small cluster (A few tens of nodes), more convenient to put the namenode, secondary nn and jobtracker on a single machine. as the cluster gets larger, these should be seperated.
* the nn has a high mem requirement as it holds file and block metadata for the entire namespace in mem.
* the secondary nn, although idle, has a comparable mem footprint to the primary when it creates a checkpoint. keeping the secondary nn backup on a different node from the nn allows recovery in the event of loss (or corruption) of all the nn metadata files.
* with a busy cluster, the jobtracker uses a lot of memory and CPU. so best to have its own node.

```Golden rules:
	* run the hdfs control scripts from the nn.
	* run the MapReduce control scripts from the jobtracker machine
	```

### Environment setting

#### memory
* def, hadoop allocates 1 GB of mem to each daemon it runs. controlled by ```HADOOP_HEAPSIZE``` setting in hadoop-env.sh
* the tasktrackers launches seperate child JVMs to run MR tasks in. take up memeory.
* the max no of map tasks that can run on the tasktracker at one time is controlled my ```mapred.tasktracker.map.tasks.maximum```
* master nodes: each of the nn and secondary nn and jobtracker daemons use 1 GB each by def
* general rule of thumb for mem allocation for nn: allow 1 GB of mem per million blocks of storage. e.g a 200-node cluster with 4TB of disk space per node, block size 128 MB and replication X3 has room for around 2 million blovks therefore nn should have 2gb

#### Java
* JAVA_HOME

	#### System logfiles

	stored in ```$HADOOP_INSTALL/``` by default. ```HADOOP_LOG_DIR``` setting in hadoop-env.sh. common choice is /var/log/hadoop. e.g. ```export HADOOP_LOG_DIR=/var/log/hadoop```

#### HDFS
	* to run hdfs, you need to designate one machine as a namenode. e.g fs.default.name is an HDFS filesystem URI whose host is the namenode's hostname or IP address. default port is 8020.
	* other important properties are those that set the storage dir for the namenode and the datanodes. ```dfs.name.dir``` specifies a list of directories where the nn stores persistent filesystem metadata (edit log and image).

	fs.default.name 		URI					file:///				The default filesyste. uri defines hostname
																		and port the nn's RPC server runs on. 8020

	dfs.name.dir 			comma-sep dir 		${hadoop.tmp.dir}/ 		list of dirs where nn stores metadata
							names 				dfs/name

	dfs.data.dir 			comma-sep dir 		${hadoop.tmp.dir}/ 		list of dirs where the datanode stores blocks.
							names 				dfs/data

	fs.checkpoint.dir   	comma-sep dir 		${hadoop.tmp.dir}/ 		list of dirs where the sec nn stores checkpoints
	    					name 				dfs/namesecondary


### Hadoop Daemon Addresses and ports

hadoop daemons generally run both an RPC server for comms between daemons and an HTTP server to provide web pages for human reading.

specifying the netwrok add as 0.0.0.0, hadoop will bind to all addresses on the machine. a port no of 0 instructs the server to start on a free port

### misc properties

#####	buffer size
hadoop uses a buffer size of 4kb for its I/O operations. best to increase this to 128kb. in iio.file.buffer.size (core-site.xml)

#####hdfs block size
defualt 64mb. many clusters use 128 mb or even 256 to ease memory pressure on the nn and to give mappers more data to work on. ```dfs.block.size``` (hdfs-site.xml)

##### Trash
moving files into this dir does not actually delete files but rather move them to trash folder. remain for min period before being permanently deleted by the system. min period set by ```fs.trash.interval```. default is 0 so files are never deleted.

can use expunge to trash, which will delete files that have been in the trash longer than their min period. ```hadoop fs -expunge```.

each user has their own .trash folder in their home dir.

### cerate users

	hadoop fs -mkdir /user/username
	hadoop fs -chown username:username /user/username
	hadoop dfsadmin -setSpaceQuota 1t /user/username			:	limits thespace in the user dir to 1 TB


### YARN Config

in yarn, ther is no jobtracker or tasktracker. instead there is a single resource manager running onnthe same machine as the HDFS nn instead (for small clusters). or on a dedicated machine.

YARN start-yarn.sh script (in sbin dir) starts the YARN daemons in the cluster. cript will start resource manager and a node manager on each machine listed in the slave file.

YARN has  ajob history server daemon that provides users with the details of the past job runs, and a web app proxy server for providing a secure way for users to access the UI provided by YARN apps

yarn-env.sh 		bash script 			environment variables that are used to run YARN

yarn-site.xml 		Hadoop config xml		configuration setting for YARN daemons, the resource manager, job history server,
											webapp proxy, and node managers


YARN resource manager address is controlled via yarn.resourcemanager.address (host-port pair).
mapreduce.framework.name must be set to yarn for the client to use YARN rather than the local job runner.
in new version mapred.local.dir == yarn.nodemanager.local-dirs which allows to specify the local disks to store intermediate data on.

YARN doe not ahve tasktrackers to serve map outputs to reduce tasks. this function then relies on shuffle handlers. these need to be enabled in yarn-site.xml by setting yarn.nodemanager.aux-services property to mapreduce.shuffle


##### important YARN daemon properties:

	yarn.resourcemanager.address 			0.0.0.0:8032				host and port the resource manager's RPC server runs on

	yarn.nodemanager.local-dirs 			/tmp/nm-local-dir			list of dirs where node managers allow containers to store 																		intermediate data. cleared when app ends.

	yarn.nodemanager.aux-services 										list of auxiliary services run by the node manager

	yarn.nodemanager.resource.memory-mb 	8192						the amount of physical mem in MB that may be allocated to 																		containers run by the node manager

	yarn.nodemanager.vmem-pmem-ratio 		2.1 						ratio of virtual to physical memory for containers


### Memory (again geeza)

rahter than specifying a fixed max number of map and reduce slots that may run on a tasktracker node at once, YARN allows applications to request an arbitary amount of memory for a task.

node managers allocate memory from a pool, so the number of tasks that are running on a particular node depends on the sum if their memorty requirements, and not simply on a fixed number of slots.

each ahdoop daemon used 1 GB, so for datanode and node manager totes to 2 GB, remainder can be dedicated to the node manager's containers by setting yarn.nodemanager.resource.memory-mb to total allocation in MB. def is 8192MB

mapreduce.map.memory.mb : determin memory options for jobs. used to specify how much mem you need for map or reduce task containers.

if a container uses more memory than it has been allocated, then it may be terminated by the node manager adn marked as failed.


### Security : Kerberos

Kerberos, a mature open-source network authentifcation protocol, to authenticate the user. it does not manage permissions. kerberos says that a user is who he says he is

Kerberos and hadoop:

1) Autentication : the client authenticates itself to the authentication server and receives a timestamped Ticket-Granting Ticket (TGT)

2) Authorization : the client uses the TGT to request a service ticket from the Ticket granting server.

3) service request : the client uses the service ticket to authenticate itself to the server that is providing the service the client is using.


the kinit command will prompt for a password. this is not required to run every a job or access hdfs since TGTs last for 10 hours

can create a Kerberos keytab file using the ```ktutil``` command. a keytab is a file that stores passwords and may be supplied to kinit with the -t option.

to enable Kerberos:
* set the ```hadoop.security.authentication``` property in core-site.xml. MIT Kerberos has to be installed seperately.
* also need to enable service-level authorization by setting hadoop.security.authorization to true in the same file
* can configure Access control Lists (ACLs) in the hadoop-policy.xml config file to control which users and groups have permission to connect to heach hadoop service

### Delegation Tokens:

isntead of using a three-step kerberos ticket exchange protocol to authenticate each call of communication between services (e.g multiple comms between hdfs and nn in read and write), hadoop uses ```delegation tokens``` to allow later authenticated access without having to contact KDC again (avoids high loads on KDC)

RPC = Remote proceedure call: is an inter-process communication that allows a computer program to cause a subroutine or procedure to execute in another address space (commonly on another computer on a shared network) without the programmer explicitly coding the details for this remote interaction

### kernal tuning (had op)
Kernel parameters should be configured in /etc/sysctl.conf so that settings survive reboots.

#### vm.swappiness
The kernel parameter vm.swappiness controls the kernel’s tendency to swap application
data from memory to disk, in contrast to discarding filesystem cache. The valid range
for vm.swappiness is 0 to 100 where higher values indicate that the kernel should be
more aggressive in swapping application data to disk, and lower values defer this be-
havior, instead forcing filesystem buffers to be discarded. Swapping Hadoop daemon
data to disk can cause operations to timeout and potentially fail if the disk is performing
other I/O operations

This is especially dangerous for HBase as Region Servers must
maintain communication with ZooKeeper lest they be marked as failed. To avoid this,
vm.swappiness should be set to 0 (zero) to instruct the kernel to never swap application
data

#### vm.overcommit_memory
Linux (and a few other Unix variants) support the ability to overcommit memory; that is, to permit more memory to be allocated than is available in physical RAM plus swap.

There are three possible settings for vm.overcommit_memory .
0 (zero)
Check if enough memory is available and, if so, allow the allocation. If there isn’t
enough memory, deny the request and return an error to the application.
1 (one)
Permit memory allocation in excess of physical RAM plus swap, as defined by
vm.overcommit_ratio . The vm.overcommit_ratio parameter is a percentage added
to the amount of RAM when deciding how much the kernel can overcommit. For
instance, a vm.overcommit_ratio of 50 and 1 GB of RAM would mean the kernel
would permit up to 1.5 GB, plus swap, of memory to be allocated before a request
failed.
2 (two)
The kernel’s equivalent of “all bets are off,” a setting of 2 tells the kernel to always
return success to an application’s request for memory. This is absolutely as weird
and scary as it sounds.

### Disk configuration

Datanodes store block data on top
of a traditional filesystem rather than on raw devices. This means all of the attributes
of the filesystem affect HDFS and MapReduce, for better or worse.

By far, the most common filesystems used in production clusters are
ext3, ext4, and xfs.

##### ext3
The third extended filesystem, or ext3, is an enhanced version of ext2. The most notable
feature of ext3 is support for journaling, which records changes in a journal or log prior
to modifying the actual data structures that make up the filesystem. It supports files up to 2 TB and a max filesystem size of 16 TB when configured with a
4 KB block size. maximum filesystem size is less of a concern with Hadoop
because data is written across many machines and many disks in the cluster

##### ext4
ext4 is extent-based, which improves sequential
performance by storing contiguous blocks together in a larger unit of storage. This is
especially interesting for Hadoop, which is primarily interested in reading and writing
data in larger blocks

Another feature of ext4 is journal checksum calculation; a feature
that improves data recoverability in the case of failure during a write.

major drawback: burn-in time. Only now is ext4
starting to see significant deployment in production systems.

```mkfs -t ext4 -m 1 -O dir_index,extent,sparse_super /dev/sdXN```
The option -t ext3 simply tells mkfs to create an ext3 filesystem while -j enables the
journal. The -m1 option is a hidden gem and sets the percentage of reserved blocks for
the superuser to 1% rather than 5%. Since no root processes should be touching data
disks, this leaves us with an extra 4% of usable disk space. With 2 TB disks, that’s up
to 82 GB

##### xfs
it’s a journaling filesystem, but the way data is organized on disk is very different.
Similar to ext4, allocation is extent-based, but its extents are within allocation groups,
each of which is responsible for maintaining its own inode table and space.

multiple processes can modify data in each allocation group without conflict


### Network Usage in Hadoop: A Review



# Administrating Hadoop

### HDFS

#### the filesystem image and edit log

all filesystem client actions are recordered in the edit log. the nn also has an in-memory representation of the filesystem metadata- it updates after the edit log has been mod. the edit log goes through flushes and sync after every action to ensure no operation is lost due to machine failure

the fsimage file is a persistent checkpoint of the filesystem metadata. it is not updated for every filesystem operation as it is a very large file. if namenode failes then the latest state of its metadata can be reconstructed in the edit log (this is safe mode).

edit log gets really big so hosting this on the secondary nn is a better option. the sec nn purppose is to produce checkpoints of the primary nn in-memory filesystem metadata. the process in which this is done:

1) the secondary asks the primary to roll its edits file, so new edits go to a new file
2) the secondary retrieves fsimage and edits from the primary (http get)
3) the secondary loads fsimage into memory, applies each operation from the edits, then creates a new consolidated fsimage file
4) the secondary sends the new fsimage back to the primary (http post).
5) the primary replaces the old fsimage with the new one and theold edits file with the new one it started in step 1.

this can be done manually using ```hadoop dfsadmin -saveNamespace```


#### Safe Mode

when the nn starts, the first thing it does is load its image file (fsimage) into memory and apply the edits from the edit log (edits). it then creates a new fsimage file (making a checkpoint).

listens to RPC and HTTP requests. however, the nn is safemode so it offers only a read-only view of the filesystem to clients.

exact block locations are stored in datanodes

during normal nn operation of the system, the nn has a map of blocks locations stored in memory. safe mode is needed to give the datanodes time to checkin to the namenode with theit block lists, so the nn can be informed of enough block locations to run the silesystem effectively

in safe mode the namenode does not issue any block-replication or delegation instructions to the datanodes

safemode is exited when the minimal replication condition is reached (99.9%) of the blocks in the whole filesystem meet thier min replication level

hadoop dfsadmin -safemode get 		: checks if safemode is on
hadoop dfsadmin -safemode enter 	: enters safemode
hadoop dfsadmin -safemode leave 	: leaves


### Audit loggin

implememnted using log4j and default at the WARN level. change the log4j properties audit to INFO for more information on all events

### Tools

#### dfsadmin : multipurpose tool for finding information about the state of the HDFS, as well as admin tasks.
				-report
				-metasave
				-safemode
				-saveNamespace

#### Filesystem check (fsck) : utility for checking the health of files in HDFS. tool looks for blocks that are missing from all datanodes, as well as under or over replicated blocks e.g

```hadoop fsck /
   hadoop fsck /user/bob/part-00007 -files -blocks -racks```

fsck retrieves all of its information from the namenode and does not coomunicate with any datanodes to retrieve any block data

##### Data block scanner

allows bad blocks to be detected and fixed before they are read by clients. the dataBlockScanner maintains a list of blocks to verify and scan them one by one for checksum errors
```dfs.datanode.scan.period.hours``` default to 504 hours (3 weeks) to perform scans.

##### Balancer

the distribution of blocks across datanodes can become unbalanced. this can affect locality for mapreduce and put greater strain on the highly utilized datanodes

the balancer is a daemon that redistributed blocks by moving them from overutilized datanodes to underutilized datanodes.

utilization of every data nodes is the ratio of used space on the node to total capacity of the node differs from the utilization of the cluster ratio of used space on the cluster to total capacity of the cluster

```start-balancer.sh```


### Monitoring

the master daemons are the most important to monitor. failure of datanodes are to be expected on large clusters so you should provide extra capacity so that the cluster can tolerate having a small % of dead nodes at any time.


### logging

hadoop daemons have a web page for changing the log level for any log4j log name found at /logLevel.

```hadoop daemonlog -setLevel jobtracker-host:50030 org.apache.hadoop.mapred.jobTracker DEBUG```
or change the log4j properties file.

#### Metrics
the HDFS and MR daemons collect information about events and measurements that are collectively known as metrics

metrics belong to a context, hadoop uses "dfs", "mapreduce", "rpc" and "jvm" contexts
metrics are configured in the conf/hadoop-metrics.properties file. default is set to null

#### Java Management Extensions (JMX)

is a standard API for monitoring and managing applications. there are several management beans (MBeans) which expose hadoop metrics to JMX-aware applications


### Maintenance
(routine admin procedures)

#### metadata backups:
if the nn's persistent metadata is lost or damanged the entire filesystem is rendered unuasable. should keep multiple copies of different ages (hour, day, week etc) to protect agasint corruption

straighforward way to make backups is to write a script to periodically archive the secondary namenode's ```previous.checkpoint``` sub-dir to an offsite location. script should test integrity of the copy. this can be done by starting a local namenode daemon and verifying that it has successfully read the fsimage andedits file into memory

#### Data backups

backup strategy is essential.
prioritise data to be backed up. highest is that which cannot be regenerated and critical to business

the distcp tool is ideal for making backups to other hdfs clusters because it can copy in parrallel

run Filesystem check (fsck) daily
run Filesystem balancer regularly


#### Commissioning and Decommissioning nodes

commissionging new nodes:
Datanodes that are permitted to connect to the namenode are specified in a file whose name is specified by the dfs.hosts preperty. file resides in nn. contains a line for each datanode specified by a network address

1) add the netwrok address of the new nodes to the include file
2) update the namenode with the new set of permitted datanodes using ```hadoop dfsadmin -refreshNodes```
3) update the jobtracker with the new set of permitted tasktrackers using ```hadoop mradmin -refreshNodes```
4) update the slaves file with the new nodes so they are included in future opps.
5) start the new datanode
6) check they appear

then run balancer

Decommissioning old nodes:

with a replication level of three eg, the chances are very high that you will lose data by simultaneously shutting down three datanodes if they are on different racks. you first have to inform the namenode you wish to take a dn out of circulation so that it can replicate blocks to other nodes.

decomm process is controlled by an ```exclude file``` which for hdfs is set by dfs.hosts.exclude property.

1) add the network address of the nodes to be decomm to the exlude file. do not update unclude file at this point
2) update the namenode with the new set of permitted datanodes, using ```hadoop dfsadmin -refreshNodes
3) update tasktrackers
5) shutdown decomm nodes
6) remove nodes from the include file then refreshNodes again.



# Misc (hadoop operations)

### Network design: 1 Gb versus 10 Gb Networks

Hadoop does not require one or the other; however, it can benefit from the additional bandwidth and lower latency of 10 Gb connectivity. So the question really becomes one of whether the benefits outweigh the
cost.
You have to consider the cost differential of the switches, the host adapters (as 10 GbE
LAN on motherboard is still not yet pervasive), optics, and even cabling, to decide if
10 Gb networking is feasible

Those that primarily run ETL-style or other high input to output data ratio MapReduce
jobs may prefer the additional bandwidth of a 10 Gb network. Analytic MapReduce
jobs—those that primarily count or aggregate numbers—perform far less network data
transfer during the shuffle phase, and may not benefit at all from such an investment.

### typical network topology

Tradtional Tree (N-tiered)
A tree may have multiple tiers, each of which brings
together (or aggregates) the branches of another tier. Hosts are connected to leaf or
access switches in a tree, which are then connected via one or more uplinks to the next
tier. The number of tiers required to build a network depends on the total number of
hosts that need to be supported


# Hive

#### installing hive

t’s handy to put Hive on your path to make it easy to launch:
% export HIVE_HOME=~/sw/apache-hive-x.y.z-bin
% export PATH=$PATH:$HIVE_HOME/bin

#### Hive shell
show tables

(The database stores its files in a directory called
metastore_db, which is relative to the location from which you ran the hive command.)

You can also run the Hive shell in noninteractive mode. The -f option runs the com‐
mands in the specified file, which is script.q in this example:
% hive -f script.q

Tables are stored as directories under Hive’s warehouse
directory, which is controlled by the hive.metastore.warehouse.dir and defaults
to /user/hive/warehouse.

#### Configure hive
Hive is configured using an XML configuration file like Hadoop’s. The file is called hive-
site.xml and is located in Hive’s conf directory.
The same directory contains hive-default.xml, which documents the properties that Hive exposes and their default values.

over-riding default config:
hive --config /Users/tom/dev/hive-conf
or change HIVE_CONF_DIR

For example, the following
command ensures buckets are populated according to the table definition (see “Buck‐
ets” on page 495):
hive> SET hive.enforce.bucketing=true;
To see the current value of any property, use SET with just the property name:
hive> SET hive.enforce.bucketing;
hive.enforce.bucketing=true

There is a precedence hierarchy to setting properties. In the following list, lower num‐
bers take precedence over higher numbers:
1. The Hive SET command
2. The command-line -hiveconf option
3. hive-site.xml and the Hadoop site files (core-site.xml, hdfs-site.xml, mapred-
site.xml, and yarn-site.xml)
4. The Hive defaults and the Hadoop default files (core-default.xml, hdfs-default.xml,
mapred-default.xml, and yarn-default.xml)

#### logging
You can find Hive’s error log on the local filesystem at ${java.io.tmpdir}/${user.name}/hive.log.
The logging configuration is in conf/hive-log4j.properties, and you can edit this file to change log levels and other logging-related settings.

#### The MEtastore

the metastore is the central repo of Hive metadata. it is divided into two pieces:
1) a service
2) backing store for the data

The solution to supporting multiple sessions (and therefore multiple users) is to use a standalone database. This configuration is referred to as a local metastore, since the metastore service still runs in the same process as the Hive service, but connects to a database running in a separate process, either on the same machine or on a remote machine.

Table 17-1. Important metastore configuration properties

Property name 				Type 					Default value 						Description
hive.metastore .
warehouse.dir 				URI 					/user/hive/ warehouse 				The directory relative to
																						fs.defaultFS where managed
																						tables are stored.

hive.metastore.uris 		Comma-separated URIs 	Not set 							If not set (the default), use an in-
																						process metastore, otherwise
																						connect to one or more remote
																						metastores, specified by a list of
																						URIs. Clients connect in a round-
																						robin fashion when there are
																						multiple remote servers.

javax.jdo.option.
ConnectionURL 				URI 					jdbc:derby:;database 				The JDBC URL of the metastore database.
													Name=metastore_db;
													create=true

javax.jdo.option.
ConnectionDriverName 		String 					org.apache.derby. 					The JDBC driver classname.
													jdbc.EmbeddedDriver

javax.jdo.option.
ConnectionUserName 			String 					APP 								The JDBC username.

javax.jdo.option.
ConnectionPassword 			String 					mine 								The JDBC password.



Schema on Read Versus Schema on Write

In a traditional database, a table’s schema is enforced at data load time. If the data being
loaded doesn’t conform to the schema, then it is rejected. This design is sometimes called
schema on write because the data is checked against the schema when it is written into
the database.
Hive, on the other hand, doesn’t verify the data when it is loaded, but rather when a
query is issued. This is called schema on read.

### tables

a hive table is logically made up of data being stored and the associted metadata describing the layout of the data in the table.
resides in hdfs, although can reside in any hadoop fs, including local or s3.

hive stored the metadata in a relational database, not in hdfs

e.g. CREATE TABLE managed_table (dummy STRING);
LOAD DATA INPATH '/user/tom/data.txt' INTO table managed_table;

will move the file hdfs://user/tom/data.txt into Hive’s warehouse directory for the
managed_table table, which is hdfs://user/hive/warehouse/managed_table

The load operation is very fast because it is just a move or re‐name within a filesystem. However, bear in mind that Hive does
not check that the files in the table directory conform to the schema declared for the table, even for managed tables. If there is a
mismatch, this will become apparent at query time, often by the query returning NULL for a missing field


CREATE EXTERNAL TABLE external_table (dummy STRING)
LOCATION '/user/tom/external_table';
LOAD DATA INPATH '/user/tom/data.txt' INTO TABLE external_table;

With the EXTERNAL keyword, Hive knows that it is not managing the data, so it doesn’t move it to its warehouse directory. Indeed, it doesn’t even check whether the external location exists at the time it is defined. This is a useful feature because it means you can
create the data lazily after creating the table.

### Partitions and buckets

hive organizes tables into partitions, a way of dividing a table into coarse-grained parts based on the value of a partition colum e.g date.

tables or partitions may be subdivided further into buckets to give extra structure to the data that may be used for more efficient queries.

Partitions are defined at table creation time using the PARTITIONED BY clause, 7 which takes a list of column definitions. For the hypothetical logfiles example, we might define a table with records comprising a timestamp and the log line itself:
```CREATE TABLE logs (ts BIGINT, line STRING)
PARTITIONED BY (dt STRING, country STRING);```

or on load

LOAD DATA LOCAL INPATH 'input/hive/partitions/file1'
INTO TABLE logs
PARTITION (dt='2001-01-01', country='GB');

```SHOW PARTITIONS logs;```

### storage formats

two dimentions that govern table storage in Hive:
* the row format
* the file format

row format dictates how rows, and the fields in a particular row, are stored. row format is defined by a SerDe "Serializer-Deserializer"


# Impala

impala uses a dedicated daemon that tuns on each datanode in the cluster. when a client runs a query it contacts an arbitary node running an impala daemon, which acts as a coordinator node for the query. the coordinator sends work to other impala daemons in the cluster and combines their results into the full result set for the query.

impala uses the Hive metastore.

# Zookeeper
distributed applications are hard to write. it is made hard because of partial failure. wheen a message is sent across the network between two nodes and the network fails, the sender does not know whether the receiver got the message. ZooKeeper cant make partial failures go away however it does not hide them. what zooKeeper does do is give you a set of tools to build distributed applications that can safely handle partial failures

Characteristics of Zookeer:
* simple
at its core, zooker is a stripped down filesystem that exposes a few simple operations, and some extra abstrations such as notifications.
* expressive
can be used to build a large lass of coordination data structures and protocls. e.g distributed queues, distributed locks etc
* Highly available
it runs on a collection of machines and is designed to be HA, applications can depend on it
* facilitates loosely coupled interactions
it can be used as a rendezvous mechanism so that processes that otherwise dont know of each others existencecan discover and interact with eachother
* it is a library
provides an open source, shared repo of implementatinos and recipes of common coordination patterns. e.g. do not have to write common protocols


### configuring zookeeper

config file called zoo.cf and placed in the conf subdirectory. can be places in /etc/zookeeper

to check whether zk is running, send the ruok command (are you ok?) to client port using nc
```echo ruok | nc localhost 2181```

### implementation

can run in two modes:

* standalone mode - there is a single zk server. good for testing
* replicated mode - used on a clster of machines. zk achieves high-availability through replication, and can provide a service as long as a majority of the machines in the ensemble (machines with zk) are up. e.g. in a 5 node ensemble, any two machines can fail and the service will still work because a marjoirty of three remain. a 6 node ensemble can also tolerate only two machines failing, since with three failures the remaining three do not constitute a majority of the six. it is usefil to have an odd number of machines in the ensemble






## exam question:
What describes the relationship between MapReduce and Hive?

Correct Answer:
Hive provides no additional capabilities to MapReduce. Hive programs are executed as MapReduce jobs via the Hive interpreter.

Hive is a framework that translates queries written in Hive QL into jobs that are executed by the MapReduce framework. Hive does not provide any functionality that isn't provided by MapReduce, but it makes some types of data operations significantly easier to perform.


Which statement most accurately describes the relationship between MapReduce and Pig?

Correct Answer:
Pig provides no additional capabilities to MapReduce. Pig programs are executed as MapReduce jobs via the Pig interpreter.

Pig is a framework that translates programs written in Pig Latin into jobs that are executed by the MapReduce framework. Pig does not provide any functionality that isn't provided by MapReduce, but it makes some types of data operations significantly easier to perform.


What is the rule governing the formatting of the underlying filesystem on slave nodes in a Hadoop cluster?

Correct Answer:
They can use different filesystems.

HDFS runs on top of a machine’s native filesystem. Each slave node runs a DataNode daemon, but those daemons are independent of each other; there is no requirement that each is running on top of the same filesystem. So you could, for example, have some slave nodes with disks formatted as ext3, some with disks formatted as ext4, and some with disks formatted as xfs. From a system administration point of view this would not be a particularly sensible thing to do, as it is much easier to administer your cluster if the machines are configured as similarly as possible, but it is certainly not a requirement that they all use the same filesystem.


Using Hadoop’s default settings, how much data will you be able to store on your Hadoop cluster if it has 12 nodes with 4TB of raw disk space per node allocated to HDFS storage?

Correct Answer:
Approximately 16TB

By default, Hadoop replicates each HDFS block three times. So if your cluster has 12 nodes, each with 4TB of disk space allocated to HDFS, you have a total of 48TB of space available. 48/3 = 16, so you can store approximately 16TB of data



You have a cluster running with the Fair Scheduler enabled and configured. You submit multiple jobs to the cluster. Each job is assigned to a pool. What are the two key points to remember about how jobs are scheduled with the Fair Scheduler?

Correct Answer:
 	Pools get a dynamically-allocated share of the available task slots (subject to additional constraints). 
 	 	Each pool’s share of task slots may change throughout the course of job execution.

 Pools are allocated their ‘fair share’ of task slots based on the total number of slots available, and also the demand in the pool -- a pool will never be allocated more slots than it needs. The pool’s share of slots may change based on jobs running in other pools; a pool with a minimum share configured, for example, may take slots away from another pool to reach that minimum share when a job runs in that pool.



 *The ResourceManager (RM) and the per-node slave NodeManager (NM) form the data-computation framework. The ResourceManager is the ultimate authority that arbitrates resources among all the applications in the system. The ResourceManager may be configured to use different schedulers to provide different algorithms for optimization.



You have configured the Fair Scheduler on your Hadoop cluster. You submit a job A, so that ONLY job A is running on the cluster. Job A requires more task resources than are available simultaneously on the cluster. Later you submit job B. Now job A and job B are running on the cluster at the same time. 

Identify two aspects of how the Fair Scheduler will arbitrate cluster resources for these two jobs?

Correct Answer:
 	When job A gets submitted, it consumes all the task resources available on the cluster.
 	When job B gets submitted, it will be allocated task resources, while job A continues to run with fewer task resources available to it.

The Fair Scheduler is designed to ensure that multiple jobs can run simultaneously on a cluster. If only one job is running on the cluster, it will consume as many task resources as it needs, up to the total capacity of the cluster. However, if another job is submitted, those jobs are then fair scheduled, so that each gets an even number of task resources.

The MRv1 Fair Scheduler allocates task resources based on available "slots". The maximum slots on a slave node is set in the configuration files. Administrators set those values based on the memory and cpu resources available for that system and taking into consideration the requirements of the average tasks run in that cluster. The JobTracker receives heartbeats to know the number of slots in use and available.

The YARN Fair Scheduler allocates containers to be used for tasks and containers are allocated based on available memory and/or vcores which are collectively called "resources". Each application may request a different amount of required resources. The maximum memory and vcores available for YARN on a particular system can be controlled in the config files by the administrator. An administrator may choose to reserve some memory or cpu resources for other processes such as HBase or Impala or even just for the local operating system. The Resource Manager receives heartbeats to know what resources are in use and available


*The NameNode receives heartbeats from DataNodes every three seconds. After five minutes (by default) without a heartbeat, the NameNode marks a DataNode as “dead.” After 10 minutes and 30 seconds, it begins to re-replicates the blocks which were held on that node.





Identify which three actions you can accomplish once you implement HDFS High Availability (HA) on your Hadoop cluster.

Correct Answer:
 	Shut one NameNode down for maintenance without halting the cluster.
 	Manually ‘fail over’ between NameNodes.
 	Automatically ‘fail over’ between NameNodes if one goes down.


*HDFS works hard to ensure that a file’s replication factor is honored. If a node leaves the cluster, any under-replicated blocks will be re-replicated elsewhere. If that node then rejoins the cluster, some blocks are over-replicated and therefore one copy of each of those blocks must be deleted. The NameNode essentially chooses a random copy of the block to be deleted (although if a rack topology script is in place, the rack placement policy will be honored). Some of the blocks may be deleted from the node which just rejoined the cluster, but that is not guaranteed, and certainly it is not the case that all blocks from that node will be deleted.



How does the NameNode know which DataNodes are currently available on a cluster?

Correct Answer:
DataNodes heartbeat in to the master on a regular basis.

DataNodes heartbeat in to the master every three seconds. When a DataNode heartbeats in to the NameNode the first time, the NameNode marks it as being available. DataNodes can be listed in a file pointed to by the dfs.hosts property, but this only lists the names of possible DataNodes. It is not a definitive list of those which are available but, rather, a list of the only machines which may be used as DataNodes if they begin to heartbeat. 

.
A client application writes a file to HDFS on your cluster. Which two metadata changes occur?

Correct Answer:
 	The metadata in RAM on the NameNode is updated
 	 	The change is written to the edits file

The NameNode metadata contains information about every file stored in HDFS. The NameNode holds the metadata in RAM for fast access, so any change is reflected in that RAM version. However, this is not sufficient for reliability, since if the NameNode crashes information on the change would be lost. For that reason, the change is also written to a log file known as the edits file. 



Assuming HDFS default settings, why does HBase write to the WAL (write-ahead log)?

Correct Answer:
To ensure that data isn’t lost in the event of a RegionServer failure