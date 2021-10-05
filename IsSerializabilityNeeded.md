# Is Serializability Needed?

As long as the devs can tune the concurrency, it is not

Most devs, are already used to manage part of concurrency. There are lots of reasons:

1. Databases have different implementation for isolation, with different guarantee, and [most don't even support serializability](http://www.bailis.org/blog/when-is-acid-acid-rarely/). Devs typically overcome with advisory locking, `FOR UPDATE/SHARE`, optimistic CC via `version` column, etc.
2. Most people also use cache to speed up read queries, and without the checking of current state. Which makes the system as a whole not serializable.
3. Big companies typically need to use microservices, to break the physical barrier to development. This makes the need for distributed transactions, in forms of choreography/orchestration appear. Both are not serializable, as I argued [here](https://github.com/aarondwi/notes/blob/main/DTXArguments.md)
4. Handling non-transactional system, such as 3rd party, already force user to think about concurrency semantics.

These big companies/systems show that even without complex CC algo, or only limited guarantee, with domain understanding, system can be made to work, meeting perf/integrity requirement:

1. Facebook only use RA + RYW. [Flighttracker](https://research.fb.com/publications/flighttracker-consistency-across-read-optimized-online-stores-at-facebook/) does RYW consistency company-wide, with [RAMP-TAO](https://engineering.fb.com/2021/08/18/core-data/ramp-tao/) for atomicity. Before those 2, facebook does fully without any kind of constraints
2. Alibaba + Ant Financial do transactions across microservices with [SEATA](https://github.com/seata/seata), which at most is just a RC system
3. Telco system, with [NDB](https://dev.mysql.com/doc/refman/8.0/en/mysql-cluster.html)/[RonDB](https://github.com/logicalclocks/rondb) which are only a RC system. [VoltDB](https://voltdb.com) has serializable per shard, 2PC cross shard. 2PC are not usually used caused it is slow. VoltDB also [XDCR](https://www.voltdb.com/blog/2021/08/xdcr/) makes it basically eventual.
