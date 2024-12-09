
```sql
test=# create index idx_dd on t_ctid_test(id);
CREATE INDEX
test=# 
test=# select pg_relation_filepath('idx_dd');
 pg_relation_filepath 
----------------------
 base/16384/49217
(1 row)
```
