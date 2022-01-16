# Regarding Multiple Isolation Levels

This is assuming the readers understand isolation levels definition with its semantic and violation.

Then, typically split to 3 big styles:

1. One with serializable read-write + optional snapshot read-only (usually for new scalable database, e.g. [FoundationDB](https://github.com/foundationDB/), [CockroachDB](cockroachlabs.com/), [Yugabyte](http://yugabyte.com/), etc). This usually implemented using MVCC/MVTO for snapshot + 2PC for read write. This is easy to use, as basically everything is serializable.
2. One with actually only one isolation level ([Singlestore](https://docs.singlestore.com/managed-service/en/getting-started-with-managed-service/about-managed-service/managed-service-faq/durability.html) only Read Committed, [Tarantool](https://www.tarantool.io/en/doc/latest/book/box/atomic/#atomic-transactions) only serializable). This is also easy to use, as the mental model is the same for all cases.
3. Those with multiple isolation level (usually traditional RDBMS). This is considered tricky to get correct, and why most new database just settled using 1 isolation level. The caveats are:

    * All built on top of locking semantic(a la [mysql's innodb](https://dev.mysql.com/doc/refman/8.0/en/innodb-introduction.html)) is easy to use. One can use both Serializable and Read Committed together without worry, as everything are just locks underneath. Beware that some databases do not put locks if not asked, and does not result in locking, see [here](https://www.percona.com/blog/what-if-mysqls-repeatable-reads-cause-you-to-lose-money/)
    * For eccentric technique based (e.g. [PostgreSQL SSI](https://www.drkp.net/papers/ssi-vldb12.pdf)), need to ensure all implementation's semantics are respected. This means if one case need serializability, if one doesn't want to think the details, all should also become serializable.

If lower isolation level is used (as are the defaults on most traditional RDBMS, or performance reasons), and a case need higher semantic (e.g. serializability), one can always materialize the conflicts and use `SELECT FOR UPDATE/SHARE`
