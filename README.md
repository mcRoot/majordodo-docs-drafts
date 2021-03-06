## Bookkeeper usage on Majordodo 

 This document describes how Majordodo takes advantage of  Bookeeper in order to implement a replicated state machine with no need of sharing any other medium other than BookKeeper.
 The global status of the system is kept on a bunch of machines (Brokers), being every change to the status first written to the log and then applied to memory.

## An introduction to Majordodo

  Majordodo is a **Distributed Resource Manager** developed at Diennea, with the main purpose to allocate resources to execute lots of concurrent tasks submitted by lots of users.

  The focus is on multi-tenancy and on the ability to handle millions of micro-tasks on hundreds of machines.
    
  The primary business requirements is the need to provide access to shared resources to lots of users while giving to each a configurable amount of resources (multitenancy) in terms of CPU, RAM, SQL/HBase databases and (distributed) filesystem-like storage.

  Majordodo has been designed to deal with thousands of users requesting executions of micro tasks: it is just like having a very big **distributed ThreadPool** with complex resource allocation facilities, which can be reconfigured at runtime.

  Guaranteed Service Level can be changed at runtime even for tasks already submitted to the system.

  Majordodo tasks can be very simple tasks, such as sending a single email message, or long running batch operations which can continue running for hours.

  When a task is assigned to a specific machine (a 'Worker' in Majordodo terms), the Broker will follow its execution, monitor it and possibly fail over the execution to another Worker in case of machine failure.

  Majordodo has been designed to deal with Worker machines which may fail at any time, a fundamental aspect in elastic deployments: to this end, in Majordodo, tasks get simply resubmitted to other machines in a transparent way, according to a service-level configuration.

  Majordodo clients submit tasks to a Broker service using a simple HTTP JSON-based API supporting transactions and the 'slots' facility.

  Workers discover the actual leader Broker and keep one and only one active TCP connection to it. A Broker-to-Worker and Broker-to-Broker protocol has been designed to support asynchronous messaging, being one connection per Worker enough for task state management. The networking stack scales up well up to hundreds of Workers with a minimal overhead on the Broker (thanks to Netty adoption).

  Majordodo is built upon **Apache BookKeeper** and **Apache ZooKeeper**, leveraging these powerful systems to implement replication and face all the usual distributed computing issues.

## Majordodo and ZooKeeper

  Majordodo Clients use ZooKeeper to discover active Brokers on the network.
  On the Broker side Majordodo exploits ZooKeeper for many situations, using it directly to address leader election, advertise the presence of services on the network and keep metadata about BookKeeper ledgers. BookKeeper, in turn, uses ZooKeeper for Bookie discovery and for ledger metadata storage.

  Among all the Brokers one is elected as 'leader', clients can connect to any of the Brokers but only the leader can change the 'status' of the system, such as accepting task submissions, and handling Workers connections.
  ZooKeeper is used to manage a shared view of the list of BookKeeper ledgers. The leader Broker creates new ledgers and drops unused ledgers, keeping on ZooKeeper the list of current ledgers.
  ZooKeeper allows the Broker to manage this kind of metadata in a safe manner by using CAS (Compare And Set) operations. Upon accessing the ledger list, the Broker can issue a conditional change operation requesting it to fail if another Broker gets in charge.

  Majordodo uses a simple leader election algorithm based on (ZooKeeper) ephemeral nodes on a given path: each Broker tries to create a node at '/majordodo/leader',and the one who succeeds is the leader.

  In order to support Brokers discovery each Broker advertises its presence on an ephemeral sequential node (such as '/majordodo/discoverypath/brokers0000007791').

## BookKeeper Usage Overview

  Apache BookKeeper is used to implement a distributed commit log with a shared-nothing architecture: no shared disk or database is needed to let the Brokers share the same view of the global status of the system.
  BookKeeper is ideal for replicating the state of Brokers, where the leader Broker keeps the global view of the status of the system in memory and logs every change to a Ledger.
  BookKeeper is used as a write-ahead commit log, that is, every change to the status is written to the log and then applied to the in-memory status.
  Other Brokers (referred to as 'followers') 'tail' the log and apply each change to their own copy of the status.
  
  Since BookKeeper ledgers can be written only once, if another Broker starts the recovery process and opens the ledger for reading it, automatically fences the previous writer so as to allow no more writes on that ledger. Thus, in case of leadership change, for instance in case of temporary network failures, the 'past' leader Broker is no longer able to log entries and, as a consequence, cannot 'change' anymore the global status of the system in memory.

## Tailing the Log and Replicating System Status

  Followers continuously read BookKeeper logs and replay each action on their local copy of system state. 

  This operation is done opening in no-recovery mode the ledger which is currently being written by the leader Broker.

  This 'tail' operation is very fast because of the internal design of the Bookie service that keeps in memory the last entries written by the client.  

 ![Brokers](image1.png)

## Snapshots

  The only shared structures between Brokers are the ZooKeeper 'filesystem' and the BookKeeper ledgers, but logs cannot be retained forever, accordingly each Broker must periodically take a snapshot of its own in-memory view of the status and persist it to disk in order to recover quickly and in order to let BookKeeper release space and resources.

  At boot the Broker loads a consistent snapshot of the status of the system at a given time and then starts to replay the log from the time (ledger offset) at which the snapshot was taken. Snapshots can be taken at any time, a snapshot is in fact contains a dump of the actual state of the system together with the actual position on BookKeeper (ledgerid + offset) 

  If no local snapshot is available the booting Broker discovers an active Broker in the network and downloads a valid snapshot from the network. For a Majordodo Broker with 500.000 active tasks the medium gzipped snapshot file size is around 20 MBytes, depending on task data payload.
  

## Ledgers Management

  The leader Broker keeps the list of active ledgers on ZooKeeper, storing for each ledger its id and creation timestamp.
  
  The update of the list of ledgers on ZooKeeper is performed as a CAS operation (update with version) to prevent multiple Brokers to concurrently modify the list.
    
  Old ledgers get deleted after a configurable amount of time, for instance when a ledger is no longer used and has been created 48 hours ago.

  In order to select the oldest ledgers, it is necessary to store the creation timestamp of each ledger. By default, at boot time,  when a 'follower' Broker remains offline for more than 48 hours, it needs to find another Broker on the network and download a snapshot, otherwise the boot will fail.

  It is also needed to store the **id of the very first Ledger** used by the system, in order to let new Brokers know that it is not possible to replay the whole history of the system by exploiting the current list of Ledgers. Indeed, if the very first ledger of system history were not present, any Broker could read all the Ledgers and build a consistent state.  
