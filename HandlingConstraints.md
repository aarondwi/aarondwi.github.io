# Handling Constraints

This is intended for large scale deployments. If a single database is enough,
please just use the features available

This is assuming fully serializable transaction is not available. If it is,
just use it and you are good.

For [I-confluence](http://www.bailis.org/papers/ca-vldb2015.pdf) data (field's length, email format, etc)
can be done anywhere, even on application level.

For non I-confluence data:

1. Foreign Key -> Immutable PK, track deleted PK to broadcast + remove all child data
2. Uniqueness -> calculate the shard, do normal single DB check there
3. Limit/check constraint (e.g. >=0) -> on per key basis, but should be on DB level
