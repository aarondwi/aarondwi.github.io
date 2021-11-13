# Current Database Concurrency Control Algorithm Landscape

This is my thoughts as why academic CC algorithms are not adopted by newest database products (and vendors).

## Background

1. Lots of production databases, implementation wise, adopt traditional 2PC/OCC + clock like + snapshot for read is adopted, and not esoteric implementations. While write may take long locks (cause distributed, fsync latency, etc), read doesn't have to suffer. And majority of workloads are read-heavy, so this works.
2. Lock based are usually easier to understand (cause already used to) and implement (cause both single node and distributed have the same pattern), for both pessimistic 2PC and OCC.

Example of latest CC used in prod DB:

1. [FoundationDB](https://www.foundationdb.org/files/fdb-paper.pdf) -> OCC 2PC, possibly with false negative but no false positive, can explicitly remove/add range.
2. [CockroachDB](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/) -> HLC OCC 2PC, like spanner
3. [Yugabyte](https://docs.yugabyte.com/latest/architecture/transactions/distributed-txns/) -> HLC OCC 2PC, like spanner
4. [TiDB](https://tikv.org/deep-dive/distributed-transaction/percolator/) -> percolator model, with backoff when meeting higher ids
5. [MongoDB](http://jepsen.io/analyses/mongodb-4.2.6) -> Snapshot isolation, HLC, OCC
6. [AtlasDB](https://palantir.github.io/atlasdb/html/transactions/transaction_protocol.html) -> percolator model, and then [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf) for serializability

And some only do best effort:

1. [Vitess](https://vitess.io/docs/overview/scalability-philosophy/)
2. Citus/Hyperscale

Some esoteric CC implementation used are:

1. [FaunaDB](https://fauna.com/blog/consistency-without-clocks-faunadb-transaction-protocol) -> an OCC variant of [Calvin](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)
2. ING Bank's Rebel -> [PSAC](https://arxiv.org/abs/1908.05940)
3. [VoltDB](https://www.voltdb.com/wp-content/uploads/2017/03/lv-technical-note-how-voltdb-does-transactions.pdf) -> [HStore](https://www.cs.cmu.edu/~pavlo/courses/fall2013/static/slides/h-store.pdf) model
4. PostgreSQL/AtlasDB -> Snapshot isolation + [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf)
5. [Couchbase](https://blog.couchbase.com/distributed-multi-document-acid-transactions/) -> [RAMP](http://www.bailis.org/papers/ramp-sigmod2014.pdf)
6. Facebook's TAO -> [RAMP-TAO](https://engineering.fb.com/2021/08/18/core-data/ramp-tao/)
7. SQL Server's [Hekaton](https://www.microsoft.com/en-us/research/publication/hekaton-sql-servers-memory-optimized-oltp-engine/)

More traditional design, which allow asking explicitly for read/write lock:

1. Traditional SQL RDBMS
2. [Neo4J](https://neo4j.com/docs/java-reference/current/transaction-management)
3. [Couchbase](https://blog.couchbase.com/distributed-multi-document-acid-transactions/)
4. [NDB/RonDB cluster](https://docs.rondb.com/intro_transactions/)
5. [Innodb cluster](https://blog.pythian.com/cluster-level-consistency-in-innodb-group-replication/)
6. [FoundationDB](https://www.foundationdb.org/files/fdb-paper.pdf) -> add/remove locks, as it is optimistic version
7. SingleStore -> [link 1](https://docs.singlestore.com/db/v7.6/en/introduction/faqs/durability/what-isolation-levels-does-singlestore-db-provide-.html), [link 2](https://docs.singlestore.com/db/v7.5/en/reference/sql-reference/data-manipulation-language-dml/select.html)

## Why?

IMHO cause other esoteric techniques, are either:

1. can't be used on distributed setting, cause too expensive
2. No explanation for management side (loading/updating ad-hoc data, add index, etc), but only focusing on performance.
3. Hard to implement correctly
4. Most real world problem have clear, easy perf target (human speed) + easily shardable per user, low contention, with high contention only on specific combined metric (total likes, etc). Even percolator easily reach 2million/s. For those cant, usually very specific only (time series, HFT, etc)
5. Focus more on higher contention, by somehow rescheduling/reordering to remove contention (but for real contention, still sequential, so not really an improvement)
6. 2PC for multi partition, only scalable per partition
7. Most academics employ non-snapshot algo, has bad perf for long running read tx
8. For non single-global tso/equiv, meaning need very careful engineering to not allow partial read
9. All their benchmarks dont include disk-write/sync and repl, only in memory. Looks really fast, but assuming failure are not correlated
10. Do not take account how to handle index update, except by also going to serializable check. This means updating data and index should be in lockstep too
11. Very wasteful on abort

Some companies, to alleviate contention issue, open separate campaign. For example, e-commerce may splits voucher redeem from the sales day itself, by asking user to manually redeem before the sales or ask user to invite others. This allows the contention to be split not into 1 min, but for example, to 3 days. This works also as easy promotions.

## About academic CCs

Notes: every non snapshot doesnt supp non blocking read! So bad for long running read only tx. But usually good enough for adhoc query by product team

1. [BCC](http://www.vldb.org/pvldb/vol9/p504-yuan.pdf) / [SSI](https://www.researchgate.net/profile/Patrick-Oneil-7/publication/220225203_Making_snapshot_isolation_serializable/links/00b49520567eace81f000000/Making-snapshot-isolation-serializable.pdf) can be used, but expensive cause need to scatter gather read range (basically back to like 2PC in distributed setting), and BCC has weird design that blocks everything for read only
2. [BOHM](https://arxiv.org/abs/1412.2324v2) is weird when need to load data cause of preallocation, but may be good cause allowing almost all parallelism. Doesn't explain indexing, management, etc. Need custom language, cause deterministic. Memory only, not including disk and replication.
3. [Orthrus](http://www.cs.umd.edu/~abadi/papers/orthrus-sigmod16.pdf)/[DORA](https://dl.acm.org/doi/10.14778/1920841.1920959) becomes weird when need result from multi partition before work -> where to do the work? Presumably will be last thread -> new bottleneck, and doesnt handle management. Need custom language, cause deterministic. Memory only, not including disk and replication.
4. [Strife](https://gunaprsd.org/assets/strife-sigmod-2020.pdf) aims to handle high contention, but has weird partitioning scheme, which takes lots of time (hundred ms), and doesnt handle management either. Need custom lang, cause deterministic. Memory only, not including disk and replication.
5. [MOCC](http://www.vldb.org/pvldb/vol10/p49-wang.pdf) means more network traffic for temperature checking, lock, etc (cant go full read/write set only during commit), but can probably be improved with caching temperature, or local only temp. Basically still OCC.
6. [Silo](http://people.csail.mit.edu/stephentu/papers/silo.pdf)/[foedus](http://www.hpl.hp.com/techreports/2015/HPL-2015-37.pdf) works both memory/dist, but those on distributed setting are using HLC/TrueTime/TSO. Variant used in most newsql
7. Even [Calvin](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf), while being deterministic, need to do reconnaissance check first (expensive if lots), and then goes to OCC like semantic
8. [HStore](https://www.cs.cmu.edu/~pavlo/courses/fall2013/static/slides/h-store.pdf) becomes full blocking on multi partition
9. [MV3C](https://www.researchgate.net/publication/311081544_Transaction_Repair_for_Multi-Version_Concurrency_Control) is expensive, cause need to communicate change back-and-forth. Can use its optimization, and do local caching to alleviate. And need custom language
10. [SSN](https://dl.acm.org/doi/10.1145/2771937.2771949) is a single global object (subject to contention). The explained fully concurrent one is far more complex, full of CAS.
11. [ermia](https://www2.cs.sfu.ca/~tzwang/ermia.pdf)/[hekaton](https://www.microsoft.com/en-us/research/publication/hekaton-sql-servers-memory-optimized-oltp-engine/) are usable. For example, out of order writing closely mimics mysql. SI is used by almost all newsql right now. Indirection used by lots in memory db. Memory only, not including disk and replication.
12. [Tictoc](https://dl.acm.org/doi/10.1145/2882903.2882935) provides nice abort reducing, like MOCC. But not strict serializable -> already more than enough. For snapshot, can utilize HLC or the like. But can cause fracture reads, cause no authority which things got read.
13. [Cicada](https://hyeontaek.com/papers/cicada-sigmod2017.pdf) is like combination of tictoc/silo/foedus/mocc/ermia/hekaton, so inherits most of its benefits or uselessness. It becomes good cause really often garbage collection, cant support long running read only tx. Memory only, not including disk and replication.
14. [PSAC](https://arxiv.org/abs/1908.05940) are very wasteful on abort
15. [Early Lock Release](https://infoscience.epfl.ch/record/152158) will complicate read semantic, will need also to go thru the log. Also cause possibility of holes in the log, as later transaction persists before older ones.
