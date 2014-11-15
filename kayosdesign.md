#Kayos Messaging and Durable Queueing

##Goal

Build a fast, low cost, fault tolerant messaging and queueing system that offers predictable performance and can take advantage of high end dedicated hardware as well as unreliable, commodity infrastructure like EC2. We want to support message de-duplication (newer versions of messages eliminate older versions) while also maintaining strict consistency (ordered synchronous delivery), causal consistency (ordered asynchronous delivery) and eventual consistency (unordered asynchonous delivery).

Low cost means we should use data locality, batching and caching strategies to make efficient use of RAM, CPU, storage IO and network IO on contemporary systems, reducing the total number of machines and resources necessary.

Fault tolerant means machines and networks can become unhealthy (slow or unresponsive) and the system can move and remove resources to regain health and also be able to quickly add new nodes to increase capacity. Recovering failed nodes and initializing new nodes should be fast and efficient. The worst time to burden the cluster with a heavy load is when adding in additional capacity, either due to replacing old nodes or just adapting to heavier traffic.

We want strong ordering and durability guarantees, to allow the easy connection to variety of consuming systems and to be able to easily reason about capabilities and semantics under both normal operating conditions and chaotic failure modes.

The deployment use case is to become an ingestion point for large amounts data generated that needs to be delivered to multiple disparate systems, such as large web scale web sites with millions of users or the internet of things that is constantly generating data to be processed. The system becomes a buffer, to accommodate realtime systems (Key/Value caching and stream analytics) as well as batch oriented (data warehousing, deep analytics) and systems between the two extremes, like RDBMSs and fulltext/search indexing.

Here will go over a non-sharded design that offers global ordered consistency, that can serve of the basis for a horizontally scalable queueing systems where only partially consistency is required.

##Technology

###De-duplication, Scheduling and Fairness

We want consumers to see messages in the order they are sent, but we also want the ability to revise messages with a new version. If messages are keyed, then any subsequent messages with the same key will overwrite previous messages, which is effectively key de-duplication. This means a consumer only has to see the most recent revision of a message, not every message version, and if a consumer has been disconnected from a long time makes it easier to catch up without the excessive work of processing every revision of every key.

This can accomplished by:

1. Assigning each unique message key a update monotonically increasing sequence number.
2. Keep a index of documents sorted by sequence value.
3. Remove an updated key's previous sequence, if it exists, when inserting the new sequence.
4. Consumers are assigned an MVCC snapshot of it's current unprocessed messages.
5. Consumers process messages in sequence order.

The supporting data structure is a Copy-On-Write (COW) Btree, which gives us O(log N) update and snapshotting space and time costs, where N is the number of unique keys.

[need diagrams here displaying the sequence index updated atomically with message de-duplication]

However this means messages are effectively reordered, breaking message ordering and potentially consistency. But consumers can still preserve strict consistency if they can commit to their representation in an isolated atomic commit. We make this possible by sending the consumers boundary markers that indicate the end of their current MVCC queue snapshot, and the consumers commit the processed messages into there representation in an atomic transaction, bringing itself into ordered consistency.

For replication, replica Kayos servers are symmetric with consumers, each downstream replica is really just a consumer of it's upstream replica. Replication can be both use chained and fanout replication. Chained to increase redundancy, and fanout to also increase consumer client capacity.

We also want to allow each consumer to proceed at it's own pace, not blocking other consumers and not using resources when inactive. Each consumer should be a lightweight addition to the master, we should be able to support a large number on consumers, concurrently or serially active, serially and any mix of the two. Each consumer will process a MVCC snapshot, and each consumer can have different snapshots with little additional overhead.

Rather than queue an ordered set of unprocessed messages separately for each consumer, the systems keeps a High Water Mark sequence for each consumer, which is a single integer that tracks the highest sequence number seen by the consumer (which can be updated strictly at the end of snapshot for full causal consistency, or any other point if only partial ordering is necessary),. Each additional consumer doesn't add load to the system except when they are actively consuming.

With this we achieve fair, de-duplicated, incremental and restartable message consumption. No message is delayed for consumption past one snapshot (a rapidly updated document doesn't delay a rarely update document beyond a single snapshot, and vice-versa).

So any mix of rapidly, rarely and never edited documents that are committed after the start of a concurrent snapshot will all be "scheduled" for the next snapshot, and multiple causal revisions are de-duplicated to the most recent revision. (note de-duplication doesn't happen at consumption time, it happens at producer commit time. But the effect is the same).

###All or Nothing Transactions

Applications that need to ensure not just message ordering, but need inter-message consistency, can use bulk all-or-nothing write transactions to provide ACID-like storage semantics, while consumers can rely on MVCC snapshotting to receive the transactions to preserve end-to-end consistency guarantees.

###Tunable Consistency

Many levels of consistency and ordering are possible depending on need an production and consumers behaviors.

    Desired Behavior   | Producers Must             | Consumers Must
    ___________________________________________________________________________
    Global Causal      | Blocked on all-or-nothing, | Process entire snapshots
	Consistency        | N-Replica commits          |
    ___________________________________________________________________________
					   
    Single Key Causal  | writers blocked on         | Process single messages
	Consitency         | N-Replica commits          |
    ___________________________________________________________________________
	
    Eventual           | Writers block on 1-Replica | Process single messages
	Consistency        | commit                     |

###Multi-Node Durable Transactions and Failover

We want to support multi-node durability N, such that N-1 replica nodes can fail and all messages are still safely held. N is the replication factor. We do this by assigning a node to be the write/production master, a node to be the read/consumer master, and flow delivery serially from write to any number of intermediate nodes before reaching the read master.

For replication factor 3, we need at least 3 nodes in a system: node A, node B, and node C.

The read/consumer node(s) will be C.
The intermediate node will be B.
The write/producer node will be A.

All producers connect and write to node A. All writes at A flow through to B, which then flow through to C. Any consumers will be only be reading from C nodes, which means it will have had to pass through B, which means it must have also passed through C.

Message flow looks like this:

    Producer(s) -> Node A -> Node B -> Node C -> Consumer(s)
	                      

For Fanout, we might look like this:

    		             		     -> Node C1 -> Consumer(s)
               	                    |
	Producer(s) -> Node A -> Node B ->  Node C2 -> Consumer(s)
               	                    |
			             			 -> Node C3 -> Consumer(s)
									 

Producers who need N factor replication before returning success will write to Node A and get success response and get back the assigned sequence number S. The producer can then take that sequence number S and poll a C node, waiting until C has committed or seen that sequence or higher. Once it gets back success, the client knows it's messages are N replica safe.

Because consumers are only ever listening on C nodes, they will never have a partially consumed message (where a message is only consumed by some consumers, instead of all consumers) due to replica loss. Any N-1 replica's can be lost before data is lost or inconsistency is possible.

To ensure a message is _durably written_ on M nodes (where M is <= replication factor N) before the message is visible, the replication of a message can be blocked on each each node (M - 1) hops from a consumer, but the consequence is all messages written after it will also be blocked until that message is durable, to preserve sequence ordering.

To increase message production capacity, multiple independent virtual shards (S1, S2, S3), of the queue will need to be created. A physical nodes can be a mix of write, intermediate or read nodes to smooth out capacity (to avoid "lumpiness" of durable data or messaging load), but the role of each node must be exclusive per shard.

Example topology of a 3 shard, triple replicated queue:

    Shard S1: Producer(s) -> Node A -> Node B -> Node C -> Consumer(s)
    Shard S2: Producer(s) -> Node B -> Node C -> Node A -> Consumer(s)
	Shard S3: Producer(s) -> Node C -> Node A -> Node B -> Consumer(s)

Such a topology means casual ordering between shards is not possible.

###Fast Failover and Recovery

The system must be able to support failing over quickly and efficiently, not just for nodes that are unresponsive, but also nodes that are slow. Because availability means more than just being able to connect to a node, the node must also be responsive, availability is better expressed as latency, sometimes as single number (average, median) or even better as percentiles (99% of requests complete with in 1 ms, 99.9% complete complete in 5ms, etc).

If an unhealthy/unresponsive node is detected we remove it from the cluster by a cluster manager. The details of a cluster manager of the clustered system is beyond the scope we will go into here, but it's possible to make such manager fault tolerant with a Paxos election of a random node by the majority of the cluster. This manager will be responsible for cluster topology as well informing producer and consumer clients which nodes to write to and read from, and cluster nodes will return an error to the client if the clients performs an action on the wrong node due to an actively shifting topology.

As messages must traverse N replicas before being considered fully committed, a single slow node before fanout in the replication chain can slow down the cluster. So we must be able to remove/kill slow nodes quickly and often (if necessary).

Simply having the clients and remaining healthy nodes route around the unhealthy node isn't enough, we must also be able to add back in nodes and get the replica's caught up quickly to bring the capacity and redundancy of the cluster back to healthy levels. So failing over can be considered something happens even in healthy clusters, small periods of slowness are the rule not the exception, especially in shared environments such as EC2.


###Master Takeover Table

When machine topologies change, it's possible for replica's to have divergent update sequences. This can happen when an upstream replica is removed/crashed and is offline for a while, and it's added back in. It may have seen updates that the new write master hasn't seen. So we need to find the divergence point of the downstream replica and roll back updates to that point.

Using a failover table, which is a list of master writer identifiers (UUIDs) mapped to the sequence a node became the master, and is generated after each startup thereafter, recording the new UUID and current sequence number.

This is failover table to for the first initialized master, it generates a UUID and adds it into the takeover table at sequence 0.

    Master ID     | Takeover Seq
	_____________________________
	0xDEADBEEF... | 0

Then 3000 messages are sent, and then the node restarted. On restart, it no it generates a new master UUID and adds the current high sequence to the takeover tableL

    Master ID     | Takeover Seq
	_____________________________
	0xDEADBEEF... | 0
	0xCAFEBABE... | 3000



So let's say we are a downstream replica node. As a downstream replica, when we connect to the upstream replica we retrieve the failover table from the compares it with our own, and find the common master ID between them. We will then rollback all changes to the sequence of the next newest entry, if it exists otherwise there is no sequence branch. If no common master is found, it completely resets it's storage.

To rollback storage and get caught back up to master, we:

1. Walk our sequence table from the divergence point, scan each message.
2. For each message key, we ask the upstream replica to send the matching messages if it's sequence is lower than the divergence point, and we write to our store.
3. If the message doesn't exist at the master, delete it from our representation.
4. After we scan everything on our side, we ask the upstream replica to send everything after the divergence point and commit to our storage.
5. We copy the failover table from the master into our store.

If all the steps succeed, we are casually consistent with master and can start receiving from upstream normally, and send them downstream or to consumers.

###Storage Engine

The durability engine used is Couchstore, a storage engine written in C and used currently in Couchbase 3.0, and evolved from the CouchDB storage engine written in Erlang. ForestDB, a more advanced storage engine based on Couchstore and is also an excellent candidate.

Both Couchstore and ForestDB have unique properties directly suitable for a message queue:

* Sequential Write IO. It's file format is a hybrid of a transaction log and storage engine. All updates are tail append, including meta data and COW B-Tree nodes. For SSDs this minimizes write amplification, for HDDs this minimizes seek operations.
* Fast Durability: All commits (inserts/updates/deletes) are batched and made durable with a single fsync of sequential writes. This includes all data, metadata, and control structures (btree nodes, etc). If a crash or power loss happens during a commit attempt, the uncommitted bytes just appear as garbage at the end of the file
* ACID: Updates aren't visible until safely committed. Offers strict ordering consistency for individual and bulk commits.
* MVCC snapshotting: The COW btree's and headers allow concurrent updates while any number of open snapshots operating on different versions of the queue state, simplifying the correct operational behavior or consumers and produces without requiring locks. 
* Pauseless compaction: When the storage file exceeds a garbage threshold, all live, unexpired data is copied into a new storage file, giving optimized locality and balancing of btrees, until caught up with main storage, which is then swapped out for the new file.

The storage technology is uniquely designed for streaming and de-duplicating data and battle proven, originally created for CouchDB, then enhanced and optimized for Couchbase.


##Conclusion

Shit be awesome yo.

