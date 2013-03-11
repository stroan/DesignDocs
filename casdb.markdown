Objectives, Benefits, Rationale
===============================

There seems to be a divide in the NoSQL DB solution space. The divide seems to relate to ease of programming vs data resilience.

There are databases like mongo and redis which are incredibly easy to program for, due to having single authorative copies of your data. You can easily read, write and update your data. They seem to largely presuppose confidence on your part of exlusively having a single concurrent read/write update cycle per key/document/whatever. They can support compare and swap semantics but they are definitely not the advertised way of using the systems. As well as this their support of data replication / sharding seems relegated to takes on traditional data partitioning with master / slave fail over. These databases are incredibly easy to program for, but have poor scaling / failover behaviors.

The other class of database is the voldemort and riak part of the market. These databases use read-repair along with consistent hashing based replication to allow for very straight forward scaling and failover behaviors. There are no slave servers, all replicas are live, serving both reads and writes. This removes the risks associated with promoting a slave in a master/slave scenario. There are very clear semantics about how many servers you can lose before data becomes unavailable, you know when a write has succeeded that it is safe and immediately available for reading. The big downside of these databases is that they are quite difficult to program for, having to build much of data structures around the read repair requirement. 

What I would like is a database which is easy to program for like the mongos/redises of the world - a database where the data model doesn't need to be crafted around the idea of having to resolving conflicting values. I would like to not have to worry about the fragile data retention guarantees of these databases, and gain some of the benefits of the distributed K/V stores like riak.

All the named databases above go through huge pains to ensure fast performance in benchmarks. I would contend that a database that provides greater guarantees, that is slower but that can scale with a similar curve to the others, would be more attractive in situations where reliability and data resilience is the name of the game. I believe that developers would also prefer to use it too, in order to not have to fight with read repair algorithms.

The purpose of this design doc is to explore the idea of a database that only performs distributed atomic operations. The primary one of interest is CAS, with which all others are implementable, albeit more slowly than if they were natively supported. It is not my contention that this can be made to compete with the redises of the world for speed, or with the voldemorts of the world for availability and failure tolerance, but with guarantees greater than the non-distributed stores, and a model simpler than the read repair stores, it may provide an attractive middle ground.

Requirements
============

  * Distributed database.
  * Master/Master replication of data.
  * Single authorative value for each key across the cluster.
  * Distributed atomic operations only.
    * Compare and swap for user state
    * ?
    * Single store only contains supports single operation
  * Resillience to server outages.
  * In the case where there is no node alive that owns a piece of data it is okay to reject both reads and writes.
  * Arbitrarily fine partitioning, using consistent hashing
  * Linear progression of data. Each "version" of data has exactly 1 ancestor.
  * Need to guarantee forward progress.

Programming model
=================

    while true
        foo, foo_version = db.get("foo")
        foo' = mutate(foo)
        result = db.put(foo', foo_version)
        if result == success then
            break
    perform_other_action(foo')

The model is similar to that of voldemort and co. You get a version of the data from the store and mutate it.
If committing the new version succeeds then you know that no changes to this state have happened between your
read and write operations, so no data has been lost. You can then use the copy of the data that was successfully
written in order to perform secondary operations, because that value is either currently in the database or is
a direct ancestor of whatever data that key now holds.

Data model
==========

All reads and writes are performed over a quorum of the replicas of the data.

Requesting data
---------------

1. Request sent to a node to get [KEY]
2. The requestee forwards the request to (N/2 + 1) of the other nodes.
3. The other nodes send back their data for [KEY].
4. A returns the highest versioned value, at [VERSION] + 1 to the client.

Updating data
-------------

In order to ensure forward progress each key can have up to two transactions associated with it, the current
transaction, and the transaction to be processed next. There is a proper ordering over transactions based on
the random identifier associated with them at creation. This ensures that in the situation where there are 
a set of transactions trying to update a single key propogating through the system, that one of them will 
successfully update the value of the key.

1. Request sent to a node to put [KEY] on top of [VERSION]
2. The requestee informs itself, and (N/2 + 1) of the other nodes to start a transaction for [KEY], 
   wrt [VERSION], with transaction id [TID] transactions ids are large integer identifiers which can be 
   assumed to be globally unique, so 128bit probably.
2. When each node receives the request to start a new transaction is does the following:
    1. Checks to see if there is an open transaction for [KEY].
       * If there is no open transaction, open one. And process the transaction as below (3)
       * If there is an open transaction, and its transaction id is less than [TID] then fail the new
         transaction.
       * If there is an open transaction, and its transaction id is greater than or equal to the [TID] 
         then check to see if there is a pending transaction for that key.
         * If there is no pending transaction, set the new transaction as the current pending 
           transaction.
         * If there is a pending transaction, and its tid is less than [TID] then fail the transaction.
         * If there is a pending transaction, and its tid is greater than or equal to [TID] then fail
           the pending transaction and set the current transaction as the pending transaction.
3. If there is an open transaction after (2), then load the data for key, and check [VERSION] against
   the version loaded. If [VERSION] is strictly greater than the value seen mark the transaction as 
   OK, otherwise send a FAIL response.
4. A gets back OKs/FAILs. If all are OK, A sends commit signal to A, B and C for that transaction id,
   otherwise it sends a ROLLBACK instruction and informs the client that the update failed.
5. When the other nodes receive a COMMIT or ROLLBACK message from the requester the data in the open
   transaction is either stored or thrown away depending. If there is a pending transaction associated
   with the one being finalized, perform step (3) for that transaction.

Examples
========

Cluster of 5 nodes. Each node knows about each other node, in a totally connected graph.

    +- [ A ] - [ B ] -+
    |    |   X   |    |
    |  [ C ]   [ D ]  |
    \     \     /     /
     ----- [ E ] -----

Action 1: Read data
-------------------

1. Request is sent to node A for "FOO"
2. A sends read request to a randomly selected subset of the nodes: A, B and E.
3. The values returned are:
       
         +-----------------------+
         | Node | Version | Data |
         +-----------------------+
         | A    | 10      | XXX  |
         | B    | 10      | XXX  |
         | E    | 12      | YYY  |
         +-----------------------+

4. The value returned is

         +----------------+
         | Version | Data |
         +----------------+
         | 13      | YYY  |
         +----------------+

Action 2: Update data
---------------------

1. Request is sent to A for "FOO", and executes exactly as in Action 1.
2. Send an update to C for "FOO" with data "AAA" and version 13.
3. C selects 3 nodes at random, C, B, and D, and attempts to open a transaction with them.
4. All nodes are able to open this transaction, because there are no other transactions on "FOO".
5. All nodes have a version for "FOO" which is less than 13, so return OK to C.
6. Upon receiving all the OKs from C, B and D, C issues a COMMIT message to C, B, and D.
7. It returns a success message to the clients.

Action 3: Update conflict
-------------------------

1. Request is sent to A for "FOO", and executes exactly as in Action 1.
2. Send an update to C for "FOO" with data "AAA" and version 13.
3. Send an update to D for "FOO" with data "BBB" and version 13.
4. C selects 3 nodes at random C, B, and E and attempts to open a transaction with them. [TID1]
5. D selects 3 nodes at random D, A, and B and attempts to open a transaction with them. [TID2]
6. C receives the request for [TID1], opens the transaction and reports OK to C.
7. D receives the request for [TID2], opens the transaction and reports OK to D.
8. E receives the request for [TID1], opens the transaction and reports OK to C.
9. A receives the request for [TID2], opens the transaction and reports OK to D.
10. Case 1: 
    * B receives the request for [TID1], opens the transaction and reports OK to C. 
    * B receives the request for [TID2]. [TID2] > [TID1], so FAIL is sent to D. 
    * C sends COMMIT to C, B and E, and informs the client of success.
    * D sends ROLLBACK to D, A, and B, and informs the client of failure.
    * "FOO" = "AAA" and the client has been informed.
11. Case 2: B receives the request for [TID2], opens the transaction and reports OK to D. B then
    receives the request for [TID1]. [TID1] < [TID2], so it is set as the pending transaction.
    * D sends COMMIT to D, A, and B, and informs the client of success.
    * B commits [TID2], and executes [TID1]. The version of the value belonging to B is no longer
      old enough to B sends FAIL to C.
    * C sends ROLLBACK to C, B and E, and informs the client of failure.
    * "FOO" = "BBB" and the client has been informed.

Action 4: Update conflict 2
---------------------------

1. Request sent to A for "FOO", and executes exactly as in Action 1.
2. Send an update to C for "FOO" with data "AAA" and version 13.
3. Send an update to D for "FOO" with data "BBB" and version 13.
4. Send an update to E for "FOO" with data "CCC" and version 13.
5. E selects nodes E, A and B, and attempts to open a transaction with them [TID1]
6. D selects nodes D, A and B, and attempts to open a transaction with them [TID2]
7. C selects nodes C, B and D, and attempts to open a transaction with them [TID3]
8. E receives the request for [TID1], opens a transaction and reports OK to E.
9. D receives the request for [TID2], opens a transaction and reports OK to D.
10. C receives the request for [TID3], and opens a transaction and reports OK to C.
11. A receives the request for [TID2], and opens a transaction and reports OK to D.
12. B receives the request for [TID3], and opens a transaction and reports OK to C.
13. A receives the request for [TID1]. It has an open transaction. [TID1] < [TID2] so [TID1] is
    set as the pending transaction on A.
14. B receives the request for [TID2]. It has an open transaction. [TID2] < [TID3] to [TID2] is
    set as the pending transaction on B.
15. D receives the request for [TID3]. It has an open transaction. [TID3] > [TID2], so it reports
    FAIL to C.
16. B receives the request for [TID1]. It has an open transaction. [TID1] < [TID3], so it tries
    to be set as the pending transaction. There is a current pending transaction. [TID2] > [TID1]
    to [TID1] is set as the pending migration and FAIL is send to D.
17. C has received all 3 responses, one of which is a FAIL, so it sends ROLLBACK to C, B and D.
    It also informs the client of failure.
18. B rolls back [TID3], and executes the pending transaction [TID1] and sends OK to E.
19. D has received all 3 responses, one of which is a FAIL, so it sends ROLLBACK to D, A and B.
    It also informs the client of failure.
20. A rolls back [TID2], and executes the pending transaction [TID1] and sends OK to E.
21. E has received all 3 responses, all of which are OK, so it sends commit to E, A and B. It
    also informs the client of success. "FOO" = "CCC" and all clients have been informed.

