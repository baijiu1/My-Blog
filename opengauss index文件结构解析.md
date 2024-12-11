
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
