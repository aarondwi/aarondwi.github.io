# Geo-distributed Database Landscape

## Typical options for geo-distributed

1. Using usual 2PL, costs latency, easiest correctness
2. Using conflict resolution, lower latency, harder correctness. But usually good enough as users are typically reside on specific area, which means source of data always originates to/from 1DC
3. Rarer options, using calvin/SLOG, immediate latency cause only 1 cross call (during log sequencing/propagation)
4. Async replication only, direct all writes to single region

## Do we really need this?

So far, nothing really need this. Even facebook directs all write to single region

Most prefer safety use repl across DC in single region, async for cross region, cause cross region latency is relatively unacceptable (typically > 100ms)

## Current Known DB implementation for Geo-distributed

1. [FaunaDB](https://fauna.com/blog/consistency-without-clocks-faunadb-transaction-protocol) -> Calvin variant, only logging need to be global. The transaction itself is the usual OCC based. But only need 1 cross region latency.
2. [Aurora multi master](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-multi-master.html#aurora-multi-master-application) -> Optimistic Concurrency Control on storage layer, has many online service layers. Service layers and storage may be on multiple availability zones inside a region
3. FoundationDB ([here](https://www.foundationdb.org/files/fdb-paper.pdf) and [here](https://quabase.sei.cmu.edu/mediawiki/index.php/FoundationDB_Data_Replication_Features)) -> Cross region async, preferably write directed to single region.
4. [Cassandra](https://research.cs.cornell.edu/ladis2009/papers/lakshman-ladis2009.pdf)/scylla -> Replication based, but no rollback. Allow other non-owning node to accept queries and replay to owning (hinted handoff), LWW conflict resolution
5. [Dynamodb global tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables_HowItWorks.html) -> CDC based, LWW conflict detection. Strongly consistent reads only for same region as writes
6. [Vitess](https://vitess.io/docs/reference/features/two-phase-commit/) -> Local, best effort via coordinator, or 2PC
7. [Cockroachdb](https://www.cockroachlabs.com/docs/cockroachcloud/architecture.html?filters=dedicated) -> 2PC over geo, really slow, but easiest to get correctness
8. [Postgres BDR](https://wiki.postgresql.org/wiki/BDR_Project) -> Custom conflict resolution based
9. [Galera](https://severalnines.com/database-blog/how-use-cluster-cluster-replication-galera-cluster) -> active-active is dangerous, doesnt have any conflict detection or 2PC
10. NDB/[RonDB](https://docs.rondb.com/rondb_global_internal/) cluster -> Sync across cluster, can add async cross region replica, RC isolation level only
11. [Consul](https://learn.hashicorp.com/tutorials/consul/reference-architecture) -> using raft in a DC/AZ, gossip for multi region
12. [VoltDB](https://www.voltdb.com/why-voltdb/activen-xdcr/) -> XDCR protocol, basically LWW + CDC/logging to kafka
