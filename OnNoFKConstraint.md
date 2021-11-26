# On No Enforcing of Foreign Key Constraints

> This note is about foreign key enforcement, as in `FOREIGN KEY column_id_here REFERENCES parent_table(target_column)` and not about putting the id in the child tables

Common reasoning of using FK enforcement (split into 2):

1. Garbage/dangling data can't be inserted
2. On update/delete of parent's key, also update/delete/set null on child's table automatically (`CASCADE / SET NULL / DO NOTHING`)

But:

1. (Against reason 1) As I argued [here](https://aarondwi.github.io/HandlingConstraints), insert can be easily managed by not allowing hard delete (a common pattern). While this may also be a read, but it doesn't need to do so in a transaction, which increases concurrency. Can also be easily cache somewhere else
2. (Against reason 2) While it is easy to use automatic update/delete cascade/etc, it is also a performance landmine, as most of the time without actually querying it people don't know how much table/rows/etc will be affected by update/delete of a row in parent table. On an extreme case, this can take lots of time and resulting in a downtime.
3. It complicates migration

    * [Vitess No FK support across machines](https://vitess.io/blog/2021-06-15-online-ddl-why-no-fk/)
    * [Github gh-ost no FK support](https://github.com/github/gh-ost/issues/331)
    * [MySQL OSC problem](https://code.openark.org/blog/mysql/the-problem-with-mysql-foreign-key-constraints-in-online-schema-changes)
