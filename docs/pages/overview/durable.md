---
title: Linearizable Consistency
keywords:
tags:
sidebar: mydoc_sidebar
permalink: durable.html
---

## Durable-LSNs

Customers can use Durable-LSN Logic with HASql and SERIALIZABLE isolation level to attain <i>Linearizable Consistency</i>, and <i>correctness in the face of network partitions</i>. Linearizability is not supported for LUA stored-procedures or triggers. We have included a partial lrl file in the distribution ('linearizable/linearizable.lrl') that shows the correct lrl switches to attain LINEARIZABLE consistency. 

### Architecture

Comdb2 is a single-master log-shipping system that uses log-shipping of physical logs to keep the replicants in-sync with the master.  A write in our system never returns control to the client until the master has replicated that write to all of the nodes in the cluster.  Replicants which fail to apply a write are marked 'incoherent' by the master.  An incoherent node isn't allowed to service any new requests.  The master is the arbiter of consistency: all replicant nodes are periodically issued a replicant-lease.  A replicant is allowed to service requests until the expiration of that lease.  If the master sees that a replicant hasn't applied an update within a period of time, it will stop issuing coherency leases to that replicant, and then defer all write requests until such time as the master is certain that the replicant's lease has expired.  The comdb2 lease system is described more fully under the [Coherency leases](transaction_model.html#Coherency leases) section.

### Anomolies

If durable-LSN logic is disabled, then when a client begins a transaction against a replicant, the replicant's current LSN defines the transaction's logical starting point. Because the replicant's current LSN can slightly trail the master's current LSN, clients may have an outdated view of the database. This isn't quite as bad as it sounds as we request that the replicant must at least be <i>coherent</i> before we allow it to service a request. So at worst we might be looking at a view of the world which is <i>lease-duration</i> milliseconds in the past. Additionally, the master blocks in distributed commit until the commit-LSN has propogated to all coherent cluster nodes, or until the commit times out. If a replicant doesn't apply an update in time, the master marks it incoherent by defering all outstanding writes until it knows that the replicant's lease has expired. So if a replicant isn't in-sync with the master, if it is coherent and allowed to handle requests, we know that the set of writes that are missing are still outstanding: the writing clients are still blocked. This means that without durable-LSN logic, while a write is outstanding a separate client may or may not see the write depending upon whether the host replicant has applied it. In the worst case, a single thread could be unlucky enough to jump from a node which has the write to a node which doesn't have the write, making it appear as if the database has gone backwards in time.

### Durable-LSN Logic

The durable-LSN scheme is pretty simple at it's core: we only allow clients to read data which has been replicated to a majority of the cluster. If a write has been written to a majority of the cluster nodes, we consider it 'durable' as it is guaranteed to be part of any sub-cluster which has at least a majority of the nodes, and we only allow a master to be elected by consent of at least a majority of the cluster nodes. When a client issues a request against a replicant, the replicant requests the most recent durable-LSN from the master. The client's sql session then begins on the replicant at a snapshot corresponding to that lsn. For the case where the current replicant hasn't reached the current durable-LSN, we simply ask that the client make the request against a different replicant. 

Under durable-LSN logic every transaction will execute at a precise logical point-in-time as determined by the master. If a commit hasn't propogated to a majority of the cluster nodes, then even though the master's physical LSN can proceed forward, the cluster's most recent durable-LSN stays the same. Any new request on any node in the cluster will begin a snapshot at the cluster's most recent durable-LSN, not (necessarily) the master's current physical LSN. Writes which aren't durable are automatically retried against different replicant. This is safe as a global table protects comdb2 from duplicate writes.

## Commentary

We thank Kyle Kingsbury and his [website](https://aphyr.com) for describing the differences between "serializable", and "linearizable" isolation levels, and for his highly amusing commentary in verifying that distributed systems correctly provide the isolation levels that they claim to provide. Alot of the entertainment in reading this blog is derived from defensive programmers or business owners who argue publicly and passionately against Kyle <i>after Kyle has shown definitively that their system fails</i>. We think it's more rational to state on the outset that <i>we might not be perfect, but we'd like to be</i>. We have tested comdb2 using Kyle's Jepsen testsuite, as well as our own tests, and the testing we've done suggests that with respect to a subset of features, we are, in fact, a linearizable system. While this sort of testing might be able to <i>suggest</i> that comdb2 is linearizable, <i>it cannot prove that comdb2 is linearizable</i>, and further, a single failure from any test-driver is sufficient to expose a flaw. If you have a reproducible testcase where comdb2 fails, <i>please forward it to us so that we can address the issue</i>. Though at times it can be embarrassing, this type of feedback is invaluable to us in that it gives us the opportunity to address our issues, and ultimately to provide our customers with a more robust and stable product.