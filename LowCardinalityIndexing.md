# Low Cardinality Indexing

This note focuses on OLTP setting, not OLAP. In OLAP setting, we can just spam more CPUs to do parallel scan/aggregate/etc. In OLTP, we can also do that, but also bottlenecked by contention for isolation, number of queries, latency target, etc.

Also assuming that the queries cant use other highly selective fields (e.g. unique key, date, etc). If it can, then the low cardinality field can be checked on `filter` step rather than `access` step.

## For simple, relatively static pattern

1. On status/enum field. Typical example is if a record is already processed or not. Either use normal index with the status/enum as first column (so combined together on search), or using partial index (e.g. `WHERE status is NULL`). Partial/Filtered index for example supported in [PostgreSQL](https://www.postgresql.org/docs/current/sql-createindex.html), [VoltDB](https://docs.voltdb.com/UsingVoltDB/ddlref_createindex.php), [Oracle](https://docs.oracle.com/cd/B19306_01/server.102/b14200/statements_5010.htm), [SQL Server](https://docs.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql), [CockroachDB](https://www.cockroachlabs.com/docs/stable/partial-indexes.html), [Yugabyte](https://docs.yugabyte.com/latest/explore/indexes-constraints/indexes-1/)
2. For relatively close in number of records, statis + known value, e.g. `VALUES IN ('A', 'B', 'C', 'D', ...)`. Basically the same as partitioning by list. Physically partitioned, so queries can directly filter by this partitioning. Supported by both [PostgreSQL](https://www.postgresql.org/docs/current/ddl-partitioning.html), [MySQL](https://dev.mysql.com/doc/refman/8.0/en/partitioning-list.html), [CockroachDB](https://www.cockroachlabs.com/docs/stable/partitioning.html), [YugaByte](https://docs.yugabyte.com/latest/explore/ysql-language-features/partitions/), [Oracle](https://docs.oracle.com/database/121/VLDBG/GUID-8928C3B0-2F83-4213-B765-EFBBF0372F64.htm), [SQL Server](https://docs.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes?), [VoltDB](https://docs.voltdb.com/UsingVoltDB/ddlref_createtable.php)

## For heavy dynamic patterns

Typical in OTA industries, as most search result has many boolean check (e.g. `HAS_SMOKE_AREA`, `HAS_AC`, `HAS_WIFI`, etc). Need to use bitmap index, and use partitioning, so do not need to use bitmask over every single record.
[Tarantool](https://www.tarantool.io/en/doc/latest/book/box/indexes/) and [Lucene](http://lucene.apache.org/) (and basically their users, [Elasticsearch](https://www.elastic.co/elasticsearch/) and [Solr](https://solr.apache.org/)) has support for this.
[Pilosa](https://github.com/pilosa/pilosa) is a full blown distributed store, specialized in bitmap indexing.

Can also use bitmap index libraries, such as [Roaring Bitmap](https://github.com/RoaringBitmap/) or those using SIMD instructions, such as [kelindar/bitmap](https://github.com/kelindar/bitmap). For few patterns to be aware of, see [here](https://medium.com/bumble-tech/bitmap-indexes-in-go-unbelievable-search-speed-bb4a6b00851)
