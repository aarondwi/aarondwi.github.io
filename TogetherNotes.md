# Notes on using Together's pattern

This is a compilation of things to know and be careful about when designing for together's pattern.

## By Difficulty to be incorporated

### Almost impossible, and probably is useless

1. DFS graph traversal (one node connection interferes with others, as in which connect to which)
2. Search Engine (interfere results with each others)

### Hard

1. PII related (security reasons)
2. Multi tenancy (security reasons)
3. Cross partition transactions -> batch only help with throughput, but may increase latency.
4. Distributed tracing -> not about storing the data, as most distributed tracing system like [Jaeger](https://www.jaegertracing.io/) and [Zipkin](https://zipkin.io/) already batch that. But about semantic of the tracing itself (spans now affect multiple traces)
5. Geospatial (need someway to lookahead, or will become unfair to some data)

### Easy

1. YCSB things, hotspot/popular item/contention-heavy behaviors, logic tx on main key/partition
2. BFS graphs traversal
3. Abstracted high level API over key value behavior.

## On incorporating / using

### Application-side

1. Start with simpler pattern, such as those with key-value access only. This typically constitute large number of requests, and very simple to batch (akin to `SELECT * FROM a_table_name WHERE some_field IN (...)`, or redis pipelines, memcache's multi_get).
2. Prefer pessimistic rather than optimistic concurrency control, so you can control how complex your logic should be without rollbacking everything.
3. Reduce allocations, as this is another source of non-useful work. See [here](https://aarondwi.github.io/WebAppsAlloc). CPU for allocations are better used for Together's logic

### Will be good for

1. Hasura/postgrest/equivalent and/or Temporal/cloudstate/dapr/equivalent, as these are abstracted API over key-value behavior
2. High contention / hotspot algo / behavior ([PSAC](https://arxiv.org/abs/1908.05940), TSO based, leaderboard, etc)
3. Materializing more (a la fdb aggregate index)
4. Sudden traffic burst (sales event), alongside [singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight)
5. Heavy ingestion (timeseries, logging, distributed tracing, etc)

### Will help a lot

1. Business level undo log, with late undo removal (close to late lock acquisition)
2. BFS lookup pattern
3. Natural scanCombineApply ([flat-combining](https://sampa.cs.washington.edu/new/papers/holt-pgas13.pdf)) behavior (skiplist, LSM, queue, stack, etc)
4. Explicit locking API, not complex CC -> easier to control
5. Scatter gather, waiting for batch to complete in parallel to reduce end-to-end latency
6. Service/DB specialization, e.g. user, relation naming, etc.
7. Splitting static and dynamic data.
