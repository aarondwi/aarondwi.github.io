# choosing DB

**99% can just use sqlite:**

It is fast enough + simple to administer. As long as no complex need, basically blogs, simple internal app, small business, etc. Now litestream also makes it easy to have backups

**99% from the 1% suffice with Postgres/MySQL. Mostly because the need of:**

1. security/auditing
2. HA/DR (with failure isolation + rto/rpo guarantee)
3. Good support for tooling + monitoring
4. Probably also a replica/secondary, for accounting/analytics

In most case, can also use these as basic search/queue/etc. This is enough for most traditional business (OTA, ERP, payment, small eCommerce, CMS, etc)

**The rest need more complex solution, for example:**

1. Social graph / knowledge graph (dgraph, neo4j, janus, nebula, etc)
2. Search engine (elastic, solr, etc)
3. Telco (Voltdb, RonDB/NDB)
4. TSDB / monitoring engine (timescale, questdb, clickhouse, prometheus, M3, kdb, influx, kudu  etc) + stream processing for IIoT (kafka, pulsar, spark, flink, etc)
5. Adtech/caching (aerospike, tarantool, voltdb, singlestore, scylla, redis, memcache, etc)
6. Geospatial (postGIS, singlestore, cockroach, tarantool, aerospike, kinetica) (for libs, H3 or S2)
7. Small analytics OLTP (HTAP) / data grid (singlestore, ksql, TiDB, kudu, geode, hazelcast, ignite, infinispan, etc)
8. Proper queuing (rocketmq, rabbitmq, nsq, etc)
9. Real DW /OLAP (clickhouse, druid, citus, singlestore, kudu, pinot, bigquery, etc)
10. Data lake solution (hadoop, delta, hudi, iceberg)

**Or even custom, such as:**

1. HFT/gaming server(disruptor, chronicle, etc) [not a database per se, but in use effectively mimics a database]
2. Scientific research (e.g. genbank)
3. Entity-resolution(tilodb)
4. Accounting-focus (tigerbeetle)
5. Machine learning embedding (embeddinghub)
6. Time travel DB (datomic, dolt)
7. Auth platform (Keto, authzed)
