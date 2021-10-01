# DB as Job Queue

Job queues, like rabbitmq, sqs, activemq, while mostly used to offload work to background, some considered it bad cause they are:

1. inflexible, you need to create your own protocol on top of it (basically state -> paid, cancelled, etc. These need to be stored separately, e.g. in DB)
2. Need to save to db, but not safe, cause outside of db tx boundary

Some prefer to use db, cause equiv DS + reliable put to queue, and optimize queue access. With typical RDBMS, this will still get ~10k qps, so fast enough. In fact, these products / companies use it (some with modifications):

1. [Facebook FOQS](https://engineering.fb.com/2021/02/22/production-engineering/foqs-scaling-a-distributed-priority-queue/)
2. [Apple's QuiCK](https://www.foundationdb.org/files/QuiCK.pdf).
3. [Segment centrifuge](https://segment.com/blog/introducing-centrifuge/)
4. [Mail ru Tarantool Queue](https://github.com/tarantool/queue)
5. [Expensify's Bedrock job queue](http://bedrockdb.com/jobs.html)
6. [Bloomberg comdb2 queue](http://www.vldb.org/pvldb/vol9/p1377-scotti.pdf)
7. [Oracle AQ](https://www.oracle.com/database/technologies/advanced-queuing.html)
8. [Sql Server service broker](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-service-broker)
9. [Eventide messagedb](https://github.com/message-db/message-db)
10. [Hasura cron](https://hasurahq.medium.com/using-postgres-for-a-robust-cron-system-db503fbc1756)/[event triggers](https://hasura.io/event-triggers/)
11. [db-scheduler](https://github.com/kagkarlsson/db-scheduler)

This also applies to some auditing/logging/equiv, if need to be transactional. The downsides are:

1. while reliably putting to queue is easy, reading the correct one to pop needs lot of work
2. Typical queue metrics (current queue length, throughput, inflight, etc) basically comes down to full scan,
or gonna become a bottleneck to track.

If the performance still can't meet the reqs, the safest solution is to tail transactions using CDC to proper queuing system. This way, data still not get lost.

Related, but other systems, still has much more use:

1. [Kafka](https://kafka.apache.org/)/equiv -> for distributed log/eventing, cdc store, etc
2. [Temporal](https://temporal.io)/equiv -> writing business logic directly, no need to explicitly use outside queue
3. [NATS](https://nats.io) -> high perf ephemeral pub/sub, can also as log with jetstream
4. [Disruptor](https://github.com/LMAX-Exchange/disruptor)/[Chronicle](https://github.com/OpenHFT/Chronicle-Queue) -> very low latency queuing for HFT, manage state byself. Can read data even before got persisted

Cause, they do not try to imitate what db do, but instead more specific pattern only
