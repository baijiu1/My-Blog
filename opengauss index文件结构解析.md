
### 总体结构

<img width="855" alt="image" src="https://github.com/user-attachments/assets/4884ad0d-e186-4aca-853d-c64bb0c781ac">

## 一层结构

![image](https://github.com/user-attachments/assets/da825980-2ffa-4628-be09-090fa2ae1554)


```sql
#查看meta块
select * from bt_metap('tab1_pkey');
#查看root page的stats
select * from bt_page_stats('tab1_pkey',1);
#查看root(leaf)页里面的内容：
select * from bt_page_items('tab1_pkey',1);
```

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

```sql
test=# select * from bt_page_items('idx_dd',1);
 itemoffset | ctid  | itemlen | nulls | vars |          data           | dead | htid  | tids 
------------+-------+---------+-------+------+-------------------------+------+-------+------
          1 | (0,2) |      16 | f     | f    | 02 00 00 00 00 00 00 00 | f    | (0,2) | 
          2 | (0,3) |      16 | f     | f    | 14 00 00 00 00 00 00 00 | f    | (0,3) | 
          3 | (0,4) |      16 | f     | f    | c8 00 00 00 00 00 00 00 | f    | (0,4) | 
          4 | (0,7) |      16 | f     | f    | d0 07 00 00 00 00 00 00 | f    | (0,7) | 
(4 rows)
```

```bash
00000000: 0000 0000 78d8 b801 0000 0000 4800 f01f  ....x.......H...
00000010: f01f 0420 0000 0000 6231 0500 0400 0000  ... ....b1......
00000020: 0100 0000 0000 0000 0100 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 f0bf  ................
00000040: 0100 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

header解析：
| page lsn           | page checksum | pd_flags | pd_lower | pd_upper | pd_special | pd_pagesize_version | pd_prune_xid |
|--------------------|-------------|----------|-----------|-----------|----------|-----------------------|------------|
|0000 0000 78d8 b801|    0000      |    0000  |   4800     |   f01f   |   f01f   |           0420        |  0000 0000  |

meta data解析(这里指向了root page的page id)：
| btm_magic | btm_version | btm_root | btm_level | btm_fastroot | btm_fastlevel | btm_last_cleanup_num_delpages | btm_last_cleanup_num_heap_tuples | btm_allequalimage |
|-----------|-------------|----------|-----------|--------------|---------------|-------------------------------|----------------------------------|-------------------|
| 6231 0500 |  0400 0000  | 0100 0000| 0000 0000 |   0100 0000  |   0000 0000   |          0000 0000            |         0000 0000 0000 0000      |         01        |

```sql
test=# select * from bt_metap('idx_oo');
 magic  | version | root | level | fastroot | fastlevel | last_cleanup_num_delpages | last_cleanup_num_tuples | allequalimage 
--------+---------+------+-------+----------+-----------+---------------------------+-------------------------+---------------
 340322 |       4 |  290 |     2 |      290 |         2 |                         0 |                      -1 | t
(1 row)
```


```c++
typedef struct BTMetaPageData
{
	uint32		btm_magic;		/* should contain BTREE_MAGIC */
	uint32		btm_version;	/* nbtree version (always <= BTREE_VERSION) */
	BlockNumber btm_root;		/* current root location uint32 */
	uint32		btm_level;		/* tree level of the root page */
	BlockNumber btm_fastroot;	/* current "fast" root location uint32 */
	uint32		btm_fastlevel;	/* tree level of the "fast" root page */
	/* remaining fields only valid when btm_version >= BTREE_NOVAC_VERSION */

	/* number of deleted, non-recyclable pages during last cleanup */
	uint32		btm_last_cleanup_num_delpages;
	/* number of heap tuples during last cleanup (deprecated) */
	float8		btm_last_cleanup_num_heap_tuples;

	bool		btm_allequalimage;	/* are all columns "equalimage"? */
} BTMetaPageData;
```



```bash
00001ff0: 0000 0000 0000 0000 0000 0000 0800 0000  ................
00002000: 0200 0000 6064 4cad 0000 0000 3000 901f  ....`dL.....0...
00002010: f01f 0420 0000 0000 e09f 2000 d09f 2000  ... ...... ... .
00002020: c09f 2000 b09f 2000 a09f 2000 909f 2000  .. ... ... ... .
00002030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00002040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

SpecialSpage指向的header解析：
| page lsn           | page checksum | pd_flags | pd_lower | pd_upper | pd_special | pd_pagesize_version | pd_prune_xid |
|--------------------|-------------|----------|-----------|-----------|----------|-----------------------|------------|
|0200 0000 6064 4cad |    0000      |    0000  |   3000   |   901f   |   f01f   |           0420        |  0000 0000  |

header后续pd_linp数组的解析，针对该索引：
| pd_linp[0] | pd_linp[1] | pd_linp[2] | pd_linp[3] | pd_linp[4] | pd_linp[5] |
|------------|------------|------------|------------|------------|------------|
| e09f 2000  | d09f 2000  | c09f 2000  | b09f 2000  | a09f 2000  | 909f 2000  |

算出来的是偏移量，真正的数据应该再加上前面的偏移量.1FF0转换十进制是8192，对应页大小，再加上pd_linp的偏移量就是真正的索引地址。



root page:
```sql
test=# select * from bt_page_stats('idx_vm',1);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     1 | l    |          6 |          0 |            16 |      8192 |      8028 |         0 |         0 |          0 |          3
(1 row)
```
bt_page_stats数据取自：
```c++
bt_page_stats
  GetBTPageStatistics
    BTPageOpaque opaque = (BTPageOpaque) PageGetSpecialPointer(page);
    ...
    stat->btpo_prev = opaque->btpo_prev;
	stat->btpo_next = opaque->btpo_next;
	stat->btpo.level = opaque->btpo.level;
	stat->btpo_flags = opaque->btpo_flags;
	stat->btpo_cycleid = opaque->btpo_cycleid;
	...

```


```bash
00003fb0: 0000 0000 0700 1000 d007 0000 0000 0000  ................
00003fc0: 0000 0000 0400 1000 c800 0000 0000 0000  ................
00003fd0: 0000 0000 0300 1000 1400 0000 0000 0000  ................
00003fe0: 0000 0000 0200 1000 0200 0000 0000 0000  ................
00003ff0: 0000 0000 0000 0000 0000 0000 0300 0000  ................
00004000: 0a                                       .
```

index tuple物理结构解析：
|     ctid     | t_info |     index data     |
|--------------|--------|--------------------|
|0000 0000 0700|  1000  |d007 0000 0000 0000 |


SpecialSpage物理结构解析(对应到BTPageOpaqueData结构体的数据，同时对应到root page)：
| btpo_prev | btpo_next | btpo_level | btpo_flags | btpo_cycleid |
|-----------|-----------|------------|------------|--------------|
| 0000 0000 | 0000 0000 | 0000 0000  |    0300    |     0000     |


```c++
typedef struct BTPageOpaqueData
{
	BlockNumber btpo_prev;		/* left sibling, or P_NONE if leftmost uint32 */
	BlockNumber btpo_next;		/* right sibling, or P_NONE if rightmost uint32 */
	uint32		btpo_level;		/* tree level --- zero for leaf pages */
	uint16		btpo_flags;		/* flag bits, see below */
	BTCycleId	btpo_cycleid;	/* vacuum cycle ID of latest split uint16 */
} BTPageOpaqueData;
```


## 二层结构
包括meta page, root page, leaf page.

![image](https://github.com/user-attachments/assets/9397b34d-553f-47f4-9b88-25c123a19d3f)

```sql
test=# create table t_idx_2(id int, name char(90));
CREATE TABLE
test=# insert into t_idx_2 values (generate_series(1,10000), md5(random()::text));
INSERT 0 10000
test=# create index idx_2 on t_idx_2(id);
CREATE INDEX
test=#
```

这是整个索引文件的开头，包含了header和meta page的信息
```bash
00000000: 0200 0000 58d0 deaf 0000 0000 4800 f01f  ....X.......H...
00000010: f01f 0420 0000 0000 6231 0500 0400 0000  ... ....b1......
00000020: 0300 0000 0100 0000 0300 0000 0100 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 f0bf  ................
00000040: 0100 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

header解析：
| page lsn           | page checksum | pd_flags | pd_lower | pd_upper | pd_special | pd_pagesize_version | pd_prune_xid |
|--------------------|-------------|----------|-----------|-----------|----------|-----------------------|------------|
|0200 0000 58d0 deaf |    0000      |    0000  |   4800     |   f01f   |   f01f   |           0420        |  0000 0000  |

meta data解析(这里指向了root page的page id)：
| btm_magic | btm_version | btm_root | btm_level | btm_fastroot | btm_fastlevel | btm_last_cleanup_num_delpages | btm_last_cleanup_num_heap_tuples | btm_allequalimage |
|-----------|-------------|----------|-----------|--------------|---------------|-------------------------------|----------------------------------|-------------------|
| 6231 0500 |  0400 0000  | 0300 0000| 0100 0000 |   0300 0000  |   0100 0000   |          0000 0000            |         0000 0000 0000 0000      |         01        |


上面的meta page指向了page id = 3的页，我们找到第三页，也就是root page，也就是0x8000前面，0x5FF0 - 0x7FF0的区间
```bash
00006000: 0200 0000 58d0 deaf 0000 0000 8800 381e  ....X.........8.
00006010: f01f 0420 0000 0000 e89f 1000 d89f 2000  ... .......... .
00006020: c89f 2000 b89f 2000 a89f 2000 989f 2000  .. ... ... ... .
00006030: 889f 2000 789f 2000 689f 2000 589f 2000  .. .x. .h. .X. .
00006040: 489f 2000 389f 2000 289f 2000 189f 2000  H. .8. .(. ... .
00006050: 089f 2000 f89e 2000 e89e 2000 d89e 2000  .. ... ... ... .
00006060: c89e 2000 b89e 2000 a89e 2000 989e 2000  .. ... ... ... .
00006070: 889e 2000 789e 2000 689e 2000 589e 2000  .. .x. .h. .X. .
00006080: 489e 2000 389e 2000 389e 2000 0000 0000  H. .8. .8. .....
...
00007e30: 0000 0000 0000 0000 0000 1d00 0100 1020  ...............
00007e40: 9b26 0000 0000 0000 0000 1c00 0100 1020  .&.............
00007e50: 2d25 0000 0000 0000 0000 1b00 0100 1020  -%.............
00007e60: bf23 0000 0000 0000 0000 1a00 0100 1020  .#.............
00007e70: 5122 0000 0000 0000 0000 1900 0100 1020  Q".............
00007e80: e320 0000 0000 0000 0000 1800 0100 1020  . .............
00007e90: 751f 0000 0000 0000 0000 1700 0100 1020  u..............
00007ea0: 071e 0000 0000 0000 0000 1600 0100 1020  ...............
00007eb0: 991c 0000 0000 0000 0000 1500 0100 1020  ...............
00007ec0: 2b1b 0000 0000 0000 0000 1400 0100 1020  +..............
00007ed0: bd19 0000 0000 0000 0000 1300 0100 1020  ...............
00007ee0: 4f18 0000 0000 0000 0000 1200 0100 1020  O..............
00007ef0: e116 0000 0000 0000 0000 1100 0100 1020  ...............
00007f00: 7315 0000 0000 0000 0000 1000 0100 1020  s..............
00007f10: 0514 0000 0000 0000 0000 0f00 0100 1020  ...............
00007f20: 9712 0000 0000 0000 0000 0e00 0100 1020  ...............
00007f30: 2911 0000 0000 0000 0000 0d00 0100 1020  )..............
00007f40: bb0f 0000 0000 0000 0000 0c00 0100 1020  ...............
00007f50: 4d0e 0000 0000 0000 0000 0b00 0100 1020  M..............
00007f60: df0c 0000 0000 0000 0000 0a00 0100 1020  ...............
00007f70: 710b 0000 0000 0000 0000 0900 0100 1020  q..............
00007f80: 030a 0000 0000 0000 0000 0800 0100 1020  ...............
00007f90: 9508 0000 0000 0000 0000 0700 0100 1020  ...............
00007fa0: 2707 0000 0000 0000 0000 0600 0100 1020  '..............
00007fb0: b905 0000 0000 0000 0000 0500 0100 1020  ...............
00007fc0: 4b04 0000 0000 0000 0000 0400 0100 1020  K..............
00007fd0: dd02 0000 0000 0000 0000 0200 0100 1020  ...............
00007fe0: 6f01 0000 0000 0000 0000 0100 0000 0820  o..............
00007ff0: 0000 0000 0000 0000 0100 0000 0200 0000  ................
```

root page的Header：
| page lsn           | page checksum | pd_flags | pd_lower | pd_upper | pd_special | pd_pagesize_version | pd_prune_xid | pd_linp[0] |
|--------------------|-------------|----------|-----------|-----------|----------|-----------------------|------------|--------------|
|0200 0000 58d0 deaf |    0000      |    0000  |   8800     |   381e   |   f01f   |           0420        |  0000 0000  | e89f 1000  |

拿e89f 1000来做为实例：
1、按照小端序排序：00109fe8
2、转换为二进制100001001111111101000
3、取低15位为偏移量，转换十六进制为1fe8
4、第三页的偏移量5FF0 + 1FE8 = 7FE8
5、通过bt_page_items('idx_2',3);可以得知第一行的数据长度为8，第二行之后为16




这是第一个page的内容，也就是对应到第一个leaf page，因为数据过长，中间的数据就用省略号代替了
```bash
00001ff0: 0000 0000 0000 0000 0000 0000 0800 0000  ................
00002000: 0200 0000 58d0 deaf 0000 0000 d405 0009  ....X...........
00002010: f01f 0420 0000 0000 0089 2000 e09f 2000  ... ...... ... .
00002020: d09f 2000 c09f 2000 b09f 2000 a09f 2000  .. ... ... ... .
00002030: 909f 2000 809f 2000 709f 2000 609f 2000  .. ... .p. .`. .
...
000025c0: 5089 2000 4089 2000 3089 2000 2089 2000  P. .@. .0. . . .
000025d0: 1089 2000 0000 0000 0000 0000 0000 0000  .. .............
```

```sql
test=# select * from bt_metap('idx_2');
 magic  | version | root | level | fastroot | fastlevel | last_cleanup_num_delpages | last_cleanup_num_tuples | allequalimage 
--------+---------+------+-------+----------+-----------+---------------------------+-------------------------+---------------
 340322 |       4 |    3 |     1 |        3 |         1 |                         0 |                      -1 | t
(1 row)

-- 查看root page，btpo_flags = 2说明这是个root page，btpo_level说明这不是最底层，最底层是0
test=# select * from bt_page_stats('idx_2',3);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     3 | r    |         28 |          0 |            15 |      8192 |      7596 |         0 |         0 |          1 |          2
(1 row)

-- 查看root page上存储的数据
test=# select * from bt_page_items('idx_2',3);
 itemoffset |  ctid  | itemlen | nulls | vars |          data           | dead | htid | tids 
------------+--------+---------+-------+------+-------------------------+------+------+------
          1 | (1,0)  |       8 | f     | f    |                         |      |      | 
          2 | (2,1)  |      16 | f     | f    | 6f 01 00 00 00 00 00 00 |      |      | 
          3 | (4,1)  |      16 | f     | f    | dd 02 00 00 00 00 00 00 |      |      | 
          4 | (5,1)  |      16 | f     | f    | 4b 04 00 00 00 00 00 00 |      |      | 
          5 | (6,1)  |      16 | f     | f    | b9 05 00 00 00 00 00 00 |      |      | 
          6 | (7,1)  |      16 | f     | f    | 27 07 00 00 00 00 00 00 |      |      | 
          7 | (8,1)  |      16 | f     | f    | 95 08 00 00 00 00 00 00 |      |      | 
          8 | (9,1)  |      16 | f     | f    | 03 0a 00 00 00 00 00 00 |      |      | 
          9 | (10,1) |      16 | f     | f    | 71 0b 00 00 00 00 00 00 |      |      | 
         10 | (11,1) |      16 | f     | f    | df 0c 00 00 00 00 00 00 |      |      | 
         11 | (12,1) |      16 | f     | f    | 4d 0e 00 00 00 00 00 00 |      |      | 
         12 | (13,1) |      16 | f     | f    | bb 0f 00 00 00 00 00 00 |      |      | 
         13 | (14,1) |      16 | f     | f    | 29 11 00 00 00 00 00 00 |      |      | 
         14 | (15,1) |      16 | f     | f    | 97 12 00 00 00 00 00 00 |      |      | 
         15 | (16,1) |      16 | f     | f    | 05 14 00 00 00 00 00 00 |      |      | 
         16 | (17,1) |      16 | f     | f    | 73 15 00 00 00 00 00 00 |      |      | 
         17 | (18,1) |      16 | f     | f    | e1 16 00 00 00 00 00 00 |      |      | 
         18 | (19,1) |      16 | f     | f    | 4f 18 00 00 00 00 00 00 |      |      | 
         19 | (20,1) |      16 | f     | f    | bd 19 00 00 00 00 00 00 |      |      | 
         20 | (21,1) |      16 | f     | f    | 2b 1b 00 00 00 00 00 00 |      |      | 
         21 | (22,1) |      16 | f     | f    | 99 1c 00 00 00 00 00 00 |      |      | 
         22 | (23,1) |      16 | f     | f    | 07 1e 00 00 00 00 00 00 |      |      | 
         23 | (24,1) |      16 | f     | f    | 75 1f 00 00 00 00 00 00 |      |      | 
         24 | (25,1) |      16 | f     | f    | e3 20 00 00 00 00 00 00 |      |      | 
         25 | (26,1) |      16 | f     | f    | 51 22 00 00 00 00 00 00 |      |      | 
         26 | (27,1) |      16 | f     | f    | bf 23 00 00 00 00 00 00 |      |      | 
         27 | (28,1) |      16 | f     | f    | 2d 25 00 00 00 00 00 00 |      |      | 
         28 | (29,1) |      16 | f     | f    | 9b 26 00 00 00 00 00 00 |      |      | 
(28 rows)

-- 查看第一页的leaf page,btpo_next指向了page id = 2的页，所以也就知道了下一个leaf page的块号，btpo_flags = 1说明这是leaf page,btpo_level说明这是最底层
test=# select * from bt_page_stats('idx_2',1);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     1 | l    |        367 |          0 |            16 |      8192 |       808 |         0 |         2 |          0 |          1
(1 row)

-- 查看第二页的leaf page，btpo_next指向了page id = 4的页，也就是下一页是page id = 4的leaf page
test=# select * from bt_page_stats('idx_2',2);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     2 | l    |        367 |          0 |            16 |      8192 |       808 |         1 |         4 |          0 |          1
(1 row)

-- 查看第四页的leaf page，btpo_next指向了第五页，btpo_prev指向了leaf page的前一页的块号
test=# select * from bt_page_stats('idx_2',4);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     4 | l    |        367 |          0 |            16 |      8192 |       808 |         2 |         5 |          0 |          1
(1 row)

-- 一直到第二十九页，btpo_next为0了，说明这是最右边的leaf page了，btpo_prev指向了上一页的块号
test=# select * from bt_page_stats('idx_2',29);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
    29 | l    |        118 |          0 |            16 |      8192 |      5788 |        28 |         0 |          0 |          1
(1 row)
```







