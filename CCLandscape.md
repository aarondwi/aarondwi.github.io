# Current Database Concurrency Control Algorithm Landscape

This is my thoughts as why academic CC algorithms are not widely adopted by newest database products (and vendors).

## Background

1. Lots of production databases, implementation wise, adopt traditional 2PC/OCC + clock like + snapshot for read is adopted, and not esoteric implementations. While write may take long locks (cause distributed, fsync latency, etc), read doesn't have to suffer. And majority of workloads are read-heavy, so this works.
2. Lock based are usually easier to understand (cause already used to) and implement (cause both single node and distributed have the same pattern), which means less bugs, for both pessimistic 2PC and OCC.
3. Even with non-advanced CC algorithms, it already meets performance requirements. The problems faced in real production usually more into schema change, capacity (memory, disk, etc), recovery, which are not addressed by most academic systems.

There are some **techniques** that are already used to go around heavy contention problem, such as:

1. Some companies open separate campaign. For example, e-commerce may splits voucher redeem from the sales day itself, by asking user to manually redeem before the sales or ask user to invite others to get more discounts. This allows the contention to be split not into 1 min, but for example, to 3 days. This works also as easy promotions.
2. Another case, if only 1 type of item going to be sold (xiaomi did this kind of flash sale back before), developers can also opt not to materialize the conflict `at all`, as they are not having any limit that should be preserved, just to record who buys the thing, has already paid, etc. (Don't know if this was the technique applied, but should work)

Example of latest CCs used in known production DB:

1. [FoundationDB](https://www.foundationdb.org/files/fdb-paper.pdf) -> OCC 2PC, possibly with false negative but no false positive, can explicitly remove/add range.
2. [CockroachDB](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/)/[Yugabyte](https://docs.yugabyte.com/latest/architecture/transactions/distributed-txns/) -> HLC OCC 2PC, like spanner
3. [TiDB](https://tikv.org/deep-dive/distributed-transaction/percolator/) -> percolator model, with backoff when meeting higher ids
4. [MongoDB](http://jepsen.io/analyses/mongodb-4.2.6) -> Snapshot isolation, HLC, OCC
5. [AtlasDB](https://palantir.github.io/atlasdb/html/transactions/transaction_protocol.html) -> percolator model, and then [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf) for serializability
6. [Vitess](https://vitess.io/docs/overview/scalability-philosophy/) and/or [Citus/Hyperscale](https://www.citusdata.com/blog/2017/11/22/how-citus-executes-distributed-transactions/) -> 2PC and/or best effort, with bit more check, like deadlock detection.
7. [FaunaDB](https://fauna.com/blog/consistency-without-clocks-faunadb-transaction-protocol) -> an OCC variant of [Calvin](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)
8. ING Bank's Rebel -> [PSAC](https://arxiv.org/abs/1908.05940)
9. [VoltDB](https://www.voltdb.com/wp-content/uploads/2017/03/lv-technical-note-how-voltdb-does-transactions.pdf) -> [HStore](https://www.cs.cmu.edu/~pavlo/courses/fall2013/static/slides/h-store.pdf) model
10. PostgreSQL -> Snapshot isolation + [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf)
11. [Couchbase](https://blog.couchbase.com/distributed-multi-document-acid-transactions/) -> [RAMP](http://www.bailis.org/papers/ramp-sigmod2014.pdf)
12. Facebook's TAO -> [RAMP-TAO](https://engineering.fb.com/2021/08/18/core-data/ramp-tao/)
13. SQL Server's [Hekaton](https://www.microsoft.com/en-us/research/publication/hekaton-sql-servers-memory-optimized-oltp-engine/)
14. [GaussDB/openGauss MOT](https://www.researchgate.net/publication/344351736_Industrial-Strength_OLTP_Using_Main_Memory_and_Many_Cores) -> [Silo](http://people.csail.mit.edu/stephentu/papers/silo.pdf)

More traditional design, which allow asking explicitly for read/write lock:

1. Traditional SQL RDBMS (`FOR SHARE / FOR UPDATE`)
2. [Neo4J](https://neo4j.com/docs/java-reference/current/transaction-management)
3. [Couchbase](https://blog.couchbase.com/distributed-multi-document-acid-transactions/)
4. [NDB/RonDB cluster](https://docs.rondb.com/intro_transactions/)
5. [Innodb cluster](https://blog.pythian.com/cluster-level-consistency-in-innodb-group-replication/)
6. [FoundationDB](https://www.foundationdb.org/files/fdb-paper.pdf) -> add/remove locks, as it is optimistic version
7. SingleStore -> [link 1](https://docs.singlestore.com/db/v7.6/en/introduction/faqs/durability/what-isolation-levels-does-singlestore-db-provide-.html), [link 2](https://docs.singlestore.com/db/v7.5/en/reference/sql-reference/data-manipulation-language-dml/select.html)

## Why?

IMHO cause other esoteric techniques, are either:

1. Can't be used on distributed setting, cause too expensive (any in-memory gain fully removed on distributed setting), or no explanation how to do so at all
2. No explanation for management side (loading/updating ad-hoc data, add index, etc), but only focusing on performance. These management side basically should also go through serializable check (or is this the only way?)
3. Hard to implement correctly, need lots of state tracking
4. Most real world problem have clear, easy perf target (human speed) + easily shardable per user, low contention, with high contention only on specific combined metric (total likes, etc). Even percolator easily reach 2million/s. For those cant, usually very specific only (time series, HFT, etc)
5. Focus more on higher contention, by somehow rescheduling/reordering to remove contention (but for real contention, still sequential, so not really an improvement, unless it is CRDT like)
6. 2PC for multi partition, only scalable per partition
7. Employ non-snapshot algo, has bad perf for long running read transactions, which are majority of workloads (but should be avoided either way for high-throughput OLTP)
8. For non single-global tso/equivalent, meaning need very careful engineering to not allow partial read
9. All their benchmarks dont include disk-write/sync and repl, only in memory. Looks really fast, but assuming failure are not correlated
10. Need static workload analysis, dynamic/ad-hoc query doesn't receive optimization
11. Very wasteful on abort
12. Does not assume dynamic transaction, which is the most typical DBMS usage (JDBC, etc)
13. Need a full rewrite, not easily adaptable to current popular database/storage engine

Which means all of them does not implement all that is needed to create a proper production ready database, and the implementations need to fill them. This means lots of behaviors not yet known, and how it will impact the design/performance after the algo got implemented

## About academic CCs

Notes: every non snapshot doesnt supp non blocking read! So bad for long running read only tx (HTAP case). But usually good enough for adhoc query by product team. And although not widely used on production database, some parts of their techniques can be used for improvement

1. [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf) (and its derivative [SGSI](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/samehe-icde2011-serializable-gsi-paper.pdf) and [RSSI](https://www.vldb.org/pvldb/vol4/p783-jung.pdf))is expensive cause need to scatter gather read range (basically back to like 2PC in distributed setting), and final check should be in critical section
2. [BOHM](https://arxiv.org/abs/1412.2324v2) cause of preallocation, gonna need to allocate all when doing adhoc insertion, etc, but may be good cause allowing almost all parallelism. Doesn't explain indexing, management, etc. Need custom language, cause deterministic. Memory only, not including disk and replication.
3. [Orthrus](http://www.cs.umd.edu/~abadi/papers/orthrus-sigmod16.pdf)/[DORA](https://dl.acm.org/doi/10.14778/1920841.1920959) becomes weird when need result from multi partition before work -> where to do the work? Presumably will be last thread -> new bottleneck, and doesnt handle management. Need custom language, cause deterministic. Memory only, not including disk and replication.
4. [Strife](https://gunaprsd.org/assets/strife-sigmod-2020.pdf) aims to handle high contention, but has heavy compute partitioning scheme, which takes lots of time (hundred ms), and doesnt handle management either. Need custom lang, cause deterministic. Memory only, not including disk and replication.
5. [MOCC](http://www.vldb.org/pvldb/vol10/p49-wang.pdf) means more network traffic for temperature checking, lock, etc (cant go full read/write set only during commit), but can probably be improved with caching temperature, or local only temp. Basically still OCC.
6. [Silo](http://people.csail.mit.edu/stephentu/papers/silo.pdf)/[foedus](http://www.hpl.hp.com/techreports/2015/HPL-2015-37.pdf) works both memory/dist, but those on distributed setting are using HLC/TrueTime/TSO. Variant used in most newsql
7. Even [Calvin](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf), while being deterministic, need to do reconnaissance check first (expensive if lots), and then goes to OCC like semantic
8. [Hstore](https://www.cs.cmu.edu/~pavlo/courses/fall2013/static/slides/h-store.pdf) becomes full blocking on multi partition
9. [MV3C](https://www.researchgate.net/publication/311081544_Transaction_Repair_for_Multi-Version_Concurrency_Control) is expensive, cause need to communicate change back-and-forth. Can use its optimization, and do local caching to alleviate. And need custom language
10. [SSN](https://dl.acm.org/doi/10.1145/2771937.2771949) is a single global object (subject to contention). The explained fully concurrent one is far more complex, full of CAS, retries, state-tracking, etc.
11. [Ermia](https://www2.cs.sfu.ca/~tzwang/ermia.pdf)/[Hekaton](https://www.microsoft.com/en-us/research/publication/hekaton-sql-servers-memory-optimized-oltp-engine/) are usable. For example, out of order writing closely mimics mysql. SI is used by almost all newsql right now. Indirection used to alleviate locks.
12. [Tictoc](https://dl.acm.org/doi/10.1145/2882903.2882935) provides nice abort reducing, like MOCC. But not strict serializable -> already more than enough. For snapshot, can utilize HLC or the like. But can cause fracture reads, cause no authority which things got read.
13. [Cicada](https://hyeontaek.com/papers/cicada-sigmod2017.pdf) is like combination of tictoc/silo/foedus/mocc/ermia/hekaton, so inherits most of its benefits, but few uselessness remains. It becomes good only with really often garbage collection to remove unneeded version (or else gonna need long traversal), so cant support long running read only tx without reducing performance. But supposedly the fastest possible from this kind of OCC (against MOCC, Silo, Foedus, etc). Most optimizations only happen on single node. Memory only, not including disk and replication.
14. [PSAC](https://arxiv.org/abs/1908.05940) runs all possible cases directly without waiting, assuming both success and failure. Very wasteful on abort
15. [Early Lock Release](https://infoscience.epfl.ch/record/152158) will complicate read semantic, will need also to go thru the log. Also cause possibility of holes in the log, as later transaction persists before older ones.
16. [Phaser/Doppel](http://pdos.csail.mit.edu/~neha/phaser.pdf) can achieve high throughput under contention, but should only be CRDT-safe, unnecessarily block reads (cause of phase synchronization between join and split), and operations can't return data (to guarantee serializability).
17. [QURO](https://db.cs.washington.edu/events/database_day/2015/slides/query_reorder.pdf) need full static analysis of all workloads, not allowing dynamic query to be also optimized (but possible to be made incremental)
18. [IC3](https://nyuscholars.nyu.edu/en/publications/scaling-multicore-databases-via-constrained-parallel-execution)/[Transaction Chopping](https://www.comp.nus.edu.sg/~cs5226/papers/xact-chopping-tods95.pdf) also need static analysis to create dependency graph. Can't dynamically/incrementally add new transaction, as it will change the analysis, unless going back to traditional 2PC. Basically based on dependency graph between either full transaction, or piece(s) of transaction, some pieces need to wait to reduce abort. All of the checks need to be done atomically, e.g. inside a critical section
19. [BCC](http://www.vldb.org/pvldb/vol9/p504-yuan.pdf) does lots of differing for read-only (the most typical query) to reduce dependency graph size. Read Check reduction done via partitioned hash-map, so reduce critical section size, which is expensive on distributed setting. Memory only, not including disk and replication.
20. [Callas/MCC](https://www.cs.cornell.edu/lorenzo/papers/Chao15Callas.pdf) is a derivative of IC3/TxChopping, but has differentiation via nexus locks (across group as typical locks, inside group via `release locks before`/ deferred lock release). The group created is mostly from same, hot transaction. In effect, it behaves as if it is single thread per group. Across groups, still behaves as if blocking, but already achieves much parallelism inside a group. But as locks release is deffered, depends on the logic (lock all then check all or lock check in steps) basically if the front one abort, gonna abort everything else, which is wasteful. And still need static analysis, to reduce the abort
21. [RAMP](http://www.bailis.org/papers/ramp-sigmod2014.pdf) is the most scalable for Read Atomic. Basically half 2PC, with context passed to each partition, to know which other partitions included in the transaction. The cost is only on latency. Programming effort can be abstracted
22. [RedBlue](https://www.usenix.org/system/files/conference/osdi12/osdi12-final-162.pdf) allow commutative ops (blue) to be executed in parallel, while non-commutative (red) one sequentially, which means only CRDT-like ops can go parallel (Close to Phaser and MCC). This needs static analysis (or developer creating generator/shadow ops manually).
23. [Transaction Healing](https://yingjunwu.github.io/papers/sigmod2016.pdf) requires static analysis, basically at runtime instead of discarding entire work, it recomputes only from the first conflict. Change checking done in a optimized way via pointer directly. Also need to be written in custom language, so it can manage the code to re-run.
