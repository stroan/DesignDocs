# Assumptions / Requirements

* Consistency: There is always a quorum of replicas with the exact same value for a given key.
* The client has complete knowledge of the layout of the network.
* The client is fail-stop.
* In the case of client failure either commit or rollback is acceptable so long as consistency
  is preserved.
* There is no corruption of data across links, there is no byzantine fault tolerance.
* Messages may be arbitrarilly delayed.


