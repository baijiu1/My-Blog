

```sql
test=# create table t_ctid_test(id int, name varchar(20));
CREATE TABLE
test=# 
test=# insert into t_ctid_test values(1, 'aaaa');
INSERT 0 1
test=# insert into t_ctid_test values(2, 'bbb');
INSERT 0 1
test=# insert into t_ctid_test values(20, 'ccc');
INSERT 0 1
test=# insert into t_ctid_test values(200, 'ddd');
INSERT 0 1
test=# update t_ctid_test set name = 'AAA' where id = 200;
UPDATE 1
test=# select * from heap_page_items(get_raw_page('t_ctid_test', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |        t_data        
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------
  1 |   8152 |        1 |     33 |    820 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000b61616161
  2 |   8120 |        1 |     32 |    822 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x0200000009626262
  3 |   8088 |        1 |     32 |    823 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x1400000009636363
  4 |   8056 |        1 |     32 |    824 |    825 |        0 | (0,5)  |       16386 |        258 |     24 |        |       | \xc800000009646464
  5 |   8024 |        1 |     32 |    825 |      0 |        0 | (0,5)  |       32770 |      10242 |     24 |        |       | \xc800000009414141
(5 rows)
```

看lp = 4和lp = 5这两条数据，对应到insert into t_ctid_test values(200, 'ddd');和update t_ctid_test set name = 'AAA' where id = 200;
可以看到：
1、lp永远是自增的
2、t_ctid指向的是最新版本的数据，且t_ctid == lp的值
3、t_xmax指向的是最新版本的t_xmin的值
infomask:
4、258 & HEAP_UPDATED = 0，这里老版本是没有这个标志位的
5、10242 & 0x2000 = 0x2000，这里的最新版本是设置了HEAP_UPDATED这个标志位的
6、2306 & 0x2000 = 0，insert语句中的是没有设置HEAP_UPDATED这个标志位
infomask2:
1、16386 & 0xE000 = 16384 (0x4000) = HEAP_HOT_UPDATED（旧记录标志）
2、32770 & 0xE000 = 32768 (0x8000) = HEAP_ONLY_TUPLE（新元组标志）


再一次更新id = 200的数据：
```sql
test=# update t_ctid_test set name = 'AAAAA' where id = 200;
UPDATE 1
test=# select * from heap_page_items(get_raw_page('t_ctid_test', 0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |         t_data         
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------------------
  1 |   8152 |        1 |     33 |    820 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000b61616161
  2 |   8120 |        1 |     32 |    822 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x0200000009626262
  3 |   8088 |        1 |     32 |    823 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x1400000009636363
  4 |   8056 |        1 |     32 |    824 |    825 |        0 | (0,5)  |       16386 |       1282 |     24 |        |       | \xc800000009646464
  5 |   8024 |        1 |     32 |    825 |    826 |        0 | (0,6)  |       49154 |       9474 |     24 |        |       | \xc800000009414141
  6 |   7984 |        1 |     34 |    826 |      0 |        0 | (0,6)  |       32770 |      10498 |     24 |        |       | \xc80000000d4141414141
(6 rows)
```
lp = 6这条数据出现了，看下是不是和上述规则对应：
1、lp永远自增
2、lp = 5成了老版本，它的t_ctid指向了lp = 6的一行数据
3、t_xmax指向的是最新版本的t_xmin的值
4、9474(t_infomask) & 0x2000，设置了HEAP_UPDATED标志位，也设置了HEAP_XMIN_COMMITTED标志位，也设置了HEAP_XACT_MASK标志位，标识事物不可见
5、最新版本：10498 & 0x2000，有设置了HEAP_UPDATED这个标志位

infomask2:
1、49154 & 0xE000 = 49152 (0xC000)

```sql
test=# delete from t_ctid_test where id = 1;
DELETE 1
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |         t_data         
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+------------------------
  1 |   8152 |        1 |     33 |    820 |    827 |        0 | (0,1)  |        8194 |        258 |     24 |        |       | \x010000000b61616161
  2 |   8120 |        1 |     32 |    822 |      0 |        0 | (0,2)  |           2 |       2306 |     24 |        |       | \x0200000009626262
  3 |   8088 |        1 |     32 |    823 |      0 |        0 | (0,3)  |           2 |       2306 |     24 |        |       | \x1400000009636363
  4 |   8056 |        1 |     32 |    824 |    825 |        0 | (0,5)  |       16386 |       1282 |     24 |        |       | \xc800000009646464
  5 |   8024 |        1 |     32 |    825 |    826 |        0 | (0,6)  |       49154 |       9474 |     24 |        |       | \xc800000009414141
  6 |   7984 |        1 |     34 |    826 |      0 |        0 | (0,6)  |       32770 |      10498 |     24 |        |       | \xc80000000d4141414141
(6 rows)
```
删除掉id = 1这条数据，可以看到：
1、t_infomask变化了，变成了258
2、t_xmax在最新的数据，lp = 6的t_xmin基础上自增了，成了827
3、258 & 0xFFF0(HEAP_XACT_MASK) = 256
3、#define HEAP_XACT_MASK			0xFFF0	/* visibility-related bits */

infomask2:
1、8194 & 0xE000 = 0x2000 = HEAP_KEYS_UPDATED（被更新或被删除）


所以得出结论：
(xmax == 0) && (ctid == lp) && ((infomask2 & HEAP2_XACT_MASK) != HEAP_KEYS_UPDATED) = 最新版本的数据
