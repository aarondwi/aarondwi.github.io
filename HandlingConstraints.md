# Handling Constraints

> This is intended for large scale deployments. If a single instance database is enough, please just use the features available

Based on [I-confluence](http://www.bailis.org/papers/ca-vldb2015.pdf), typical constraints can be split into 2:

1. I-Confluence data, such as field's length, email format, enum, etc. This can be done anywhere, as if the data is valid on application layer, it is valid on database layer
2. Non I-confluence data, all constraint handling should be on database layer:
    1. Foreign Key ->
        * If serializable transaction is supported, use it with `SELECT * FROM parent_table WHERE id=?`, and you are good
        * If serializable transacation is not supported, use immutable PK and prefer not to hard delete it. If really needed to delete, can track deleted PK to broadcast + remove all child data
    2. Uniqueness -> Calculate the shard, do normal single DB check there. Even when cross-shard serializable distributed transaction not supported, sharding is typically deployed, and only 1 primary for each shard/key
    3. Limit/check constraint (e.g. >=0) ->
        * If hard update (direct SET), can be done anywhere, cause basically is I-Confluence
        * If not, such as INCR/DECR/etc, same as handling uniqueness
