# Multi-Tenant Background Jobs Fairness

This all only matters if lots of jobs are already queued. If only few, no fairness is okay, as systems currently not overloaded

1. Each tenant have its own queue, then has a global queue that tracks queue that has items. Worker gonna split the work (by randomize which queue to take), then hold exclusive access to it. This guarantees enough fairness, contentions only on selecting which queue to which workers, and when deleting the trackers. This pattern is the only option when need to handle billion of small queues, i.e. tenants. Like [apple's QuiCK](https://www.foundationdb.org/files/QuiCK.pdf).
2. Big tenants have their own dedicated queue, smaller ones share (usually by using priority). Workers gonna access specific queues only (coordinated via distlock). This guarantees enough fairness, almost no contentions on hot path.
3. By tracking inflight number per tenant, effectively trying as correct as possible. Workers poll inflight numbers, then go checking the items. Lots of contentions around the tracking number, and hard to look which items to process next. Can also be a variant in which instead of workers check + poll, workers just poll. Another background workers put from smaller queue to worker queue only by the allowed amount.
4. Each tenant put into their designated queue. Workers round robin poll all the queue, with pre-generated randomized order a la [SWIM](https://research.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf), and take a batch of work
