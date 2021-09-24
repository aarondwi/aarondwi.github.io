# DB as Job Queue

Job queues, like rabbitmq, sqs, activemq, while mostly used to offload work to background, some considered it bad cause they are:

1. inflexible, you need to create your own protocol on top of it (basically state -> paid, cancelled, etc. These need to be stored separately, e.g. in DB)
2. Need to save to db, but not safe, cause outside of db tx boundary

Some prefer to use db, cause equiv DS (btree/skiplist, not LSM. LSM will cause the read to be so heavy) + reliable put to queue, and optimize queue access. With typical RDBMS, this will still get ~10k qps, so fast enough. In fact, these products / companies use it (some with modifications):

1. Facebook FOQS
2. Apple fdb QuiCK
3. Segment centrifuge
4. Mail ru tarantool
5. Expensify bedrock job queue
6. Bloomberg comdb2 queue
7. Oracle AQ
8. Sql server service broker
9. Eventide messagedb
10. Hasura cron/event triggers
11. db-scheduler

This also applies to some auditing/logging/equiv, if need to be transactional. The downsides are:

1. while reliably putting to queue is easy, reading the correct one to pop needs lot of work
2. Typical queue metrics (current queue length, throughput, inflight, etc) basically comes down to full scan,
or gonna become a bottleneck to track.

If the performance still can't meet the reqs, the safest solution is to tail transactions using CDC to proper queuing system. This way, data still not get lost.

Related, but other systems, still has much more use:

1. Kafka/equiv -> for distributed log/eventing, cdc store, etc
2. Temporal/equiv -> writing business logic directly, no need to explicitly use outside queue
3. NATS -> high perf ephemeral pub/sub, can also as log with jetstream
4. Disruptor/chronicle -> very low latency queuing, manage state byself. Can read data even before got persisted

Cause, they do not try to imitate what db do, but instead more specific pattern only
