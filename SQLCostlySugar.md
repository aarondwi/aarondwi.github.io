# SQL is costly sugar for OLTP

## Why?

1. OLTP has fixed access, so sql parsing/planning is a waste (parsing + planning can take up to few hundred microseconds per query)
2. SQL usage trends also makes it hard to secure db (cause logic not on db), string parsing allowing injection, permission not on function level, etc
3. This trend also make concurrency management harder (cause longer window)
4. Also encourage user to create subpar design, not well thought, because so dynamic

## What SQL got right

1. Easily evolvable API, all just a single long string. This also makes it easy to implement lots of server side function, such as `coalesce`, `now`, date-related function, etc, without the need to do read first over network
2. Composability. Everything is just a table

## DB favoring real programming language from SQL

1. [FDB Record Layer](https://github.com/foundationdb/fdb-record-layer/)
2. [Fauna](https://fauna.com)
3. [Palantir's AtlasDB](https://github.com/palantir/atlasdb)

## some other resources

1. [Against SQL](https://scattered-thoughts.net/writing/against-sql/)
2. [Some Opinionated SQL takes](https://blog.nelhage.com/post/some-opinionated-sql-takes/)
3. [RonDB internals](https://docs.rondb.com/rondb_internals/)
4. [Linkedin's Liquid](https://engineering.linkedin.com/blog/2020/liquid--the-soul-of-a-new-graph-database--part-2)
5. [FDB Record Layer](https://foundationdb.github.io/fdb-record-layer/)
