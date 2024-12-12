
### 总体结构

<img width="855" alt="image" src="https://github.com/user-attachments/assets/4884ad0d-e186-4aca-853d-c64bb0c781ac">

## 零层结构

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


## 一层结构
包括meta page, root page, leaf page.

![image](https://github.com/user-attachments/assets/9397b34d-553f-47f4-9b88-25c123a19d3f)



<img width="829" alt="image" src="https://github.com/user-attachments/assets/299269c9-b6c8-4689-bcb8-ff24c01cfe95" />



```sql
test=# create table t_idx_2(id int, name char(90));
CREATE TABLE
test=# insert into t_idx_2 values (generate_series(1,10000), md5(random()::text));
INSERT 0 10000
test=# create index idx_2 on t_idx_2(id);
CREATE INDEX
test=#
```

#### header & meta data

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


#### root page

上面的meta page指向了page id = 3的页，我们找到第三页，也就是root page，也就是0x8000前面，0x5FF0 - 0x7FF0的区间
```bash
00006000: 0200 0000 58d0 deaf 0000 0000 8800 381e  ....X.........8.
00006010: f01f 0420 0000 0000 e89f 1000 d89f 2000  ... .......... .
00006020: c89f 2000 b89f 2000 a89f 2000 989f 2000  .. ... ... ... .
00006030: 889f 2000 789f 2000 689f 2000 589f 2000  .. .x. .h. .X. .
00006040: 489f 2000 389f 2000 289f 2000 189f 2000  H. .8. .(. ... .
...
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
1. 按照小端序排序：00109fe8
2. 转换为二进制100001001111111101000
3. 取低15位为偏移量，转换十六进制为1fe8
4. 第三页的偏移量5FF0 + 1FE8 = 7FD8
5. 通过bt_page_items('idx_2',3);可以得知第一行的数据长度为8，也就是：0000 0100 0000 0820，第二行之后为16，也就是：6f01 0000 0000 0000 1020 0100 0200 0000
6. 6f01 0000 0000 0000 0000 0200 0100 1020 这行数据，指向的是下一层的leaf page的最大值。同理dd02 0000 0000 0000 1020 0100 0400 0000指向的是第二页leaf page的最大值
7. 第一条为空，是因为这个leaf page是最左边的PAGE，不存最小值。对于有右leaf page的leaf page，第一条存储的heap item为该页的右链路。


ctid = （2，5） 表示此行存储在表文件的第2个数据叶的第5个相对位置

```sql
-- 查看root page上存储的数据
test=# select * from bt_page_items('idx_2',3);
 itemoffset |  ctid  | itemlen | nulls | vars |          data           | dead | htid | tids 
------------+--------+---------+-------+------+-------------------------+------+------+------
          1 | (1,0)  |       8 | f     | f    |                         |      |      | 
          2 | (2,1)  |      16 | f     | f    | 6f 01 00 00 00 00 00 00 |      |      | 
          3 | (4,1)  |      16 | f     | f    | dd 02 00 00 00 00 00 00 |      |      | 
	  ...
         28 | (29,1) |      16 | f     | f    | 9b 26 00 00 00 00 00 00 |      |      | 
(28 rows)
```

具体index tuple内容:
|      ctid      | t_info |     index data     |
|----------------|--------|--------------------|
| 0000 0200 0100 |  1020  |6f01 0000 0000 0000 |




#### leaf page


###### 第一个leaf page
这是第一个page的内容，也就是对应到第一个leaf page，因为数据过长，中间的数据就用省略号代替了
```bash
00002000: 0200 0000 58d0 deaf 0000 0000 d405 0009  ....X...........
00002010: f01f 0420 0000 0000 0089 2000 e09f 2000  ... ...... ... .
00002020: d09f 2000 c09f 2000 b09f 2000 a09f 2000  .. ... ... ... .
00002030: 909f 2000 809f 2000 709f 2000 609f 2000  .. ... .p. .`. .
...
00002900: 0000 0500 0100 1020 6f01 0000 0000 0000  ....... o.......
00002910: 0000 0500 2900 1000 6e01 0000 0000 0000  ....)...n.......
00002920: 0000 0500 2800 1000 6d01 0000 0000 0000  ....(...m.......
...
00003fd0: 0000 0000 0200 1000 0200 0000 0000 0000  ................
00003fe0: 0000 0000 0100 1000 0100 0000 0000 0000  ................
00003ff0: 0000 0000 0200 0000 0000 0000 0100 0000  ................
```

第一个leaf page的Header：
| page lsn           | page checksum | pd_flags | pd_lower | pd_upper | pd_special | pd_pagesize_version | pd_prune_xid | pd_linp[0] |
|--------------------|-------------|----------|-----------|-----------|----------|-----------------------|------------|--------------|
|0200 0000 58d0 deaf |    0000      |    0000  |   d405     |   0009   |   f01f   |           0420        |  0000 0000  | 0089 2000  |


拿0089 2000来做为实例：
1. 按照小端序排序：00208900
2. 转换为二进制1000001000100100000000
3. 取低15位为偏移量，转换十六进制为0900
4. 第三页的偏移量2000 + 0900 = 2900
5. 通过bt_page_items('idx_2',1);可以看到2900存的就是6F 01这个数据，这是第一页，也就是1FF0 - 3FF0区间的最大值的偏移量，之后就是从最小值开始一直到6E 01

```sql
test=# select * from bt_page_items('idx_2',1);
 itemoffset |  ctid  | itemlen | nulls | vars |          data           | dead |  htid  | tids 
------------+--------+---------+-------+------+-------------------------+------+--------+------
          1 | (5,1)  |      16 | f     | f    | 6f 01 00 00 00 00 00 00 |      |        | 
          2 | (0,1)  |      16 | f     | f    | 01 00 00 00 00 00 00 00 | f    | (0,1)  | 
          3 | (0,2)  |      16 | f     | f    | 02 00 00 00 00 00 00 00 | f    | (0,2)  | 
	...
        366 | (5,40) |      16 | f     | f    | 6d 01 00 00 00 00 00 00 | f    | (5,40) | 
        367 | (5,41) |      16 | f     | f    | 6e 01 00 00 00 00 00 00 | f    | (5,41) | 
(367 rows)
```

###### 第二个leaf page

```sql
-- 查看第一页的leaf page,btpo_next指向了page id = 2的页，所以也就知道了下一个leaf page的块号，btpo_flags = 1说明这是leaf page,btpo_level说明这是最底层
test=# select * from bt_page_stats('idx_2',1);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     1 | l    |        367 |          0 |            16 |      8192 |       808 |         0 |         2 |          0 |          1
(1 row)

test=# select * from bt_page_items('idx_2',2);

 itemoffset |  ctid   | itemlen | nulls | vars |          data           | dead |  htid   | tids 
------------+---------+---------+-------+------+-------------------------+------+---------+------
          1 | (11,1)  |      16 | f     | f    | dd 02 00 00 00 00 00 00 |      |         | 
          2 | (5,42)  |      16 | f     | f    | 6f 01 00 00 00 00 00 00 | f    | (5,42)  | 
          3 | (5,43)  |      16 | f     | f    | 70 01 00 00 00 00 00 00 | f    | (5,43)  | 
	...
        365 | (11,15) |      16 | f     | f    | da 02 00 00 00 00 00 00 | f    | (11,15) | 
        366 | (11,16) |      16 | f     | f    | db 02 00 00 00 00 00 00 | f    | (11,16) | 
        367 | (11,17) |      16 | f     | f    | dc 02 00 00 00 00 00 00 | f    | (11,17) | 
(367 rows)
```

第一页的最大值等于第二页的最小起始值


###### 倒数第二页

```sql
test=# select * from bt_page_items('idx_2',28);

 itemoffset |   ctid   | itemlen | nulls | vars |          data           | dead |   htid   | tids 
------------+----------+---------+-------+------+-------------------------+------+----------+------
          1 | (152,1)  |      16 | f     | f    | 9b 26 00 00 00 00 00 00 |      |          | 
          2 | (146,27) |      16 | f     | f    | 2d 25 00 00 00 00 00 00 | f    | (146,27) | 
```

###### 最后一页

最后一页不含右leaf page的最小值了，第一条就是起始ctid (即items)


```bash
0003a000: 0200 0000 58d0 deaf 0000 0000 f001 9018  ....X...........
0003a010: f01f 0420 0000 0000 e09f 2000 d09f 2000  ... ...... ... .
...
0003bfc0: 0000 9800 0500 1000 9d26 0000 0000 0000  .........&......
0003bfd0: 0000 9800 0400 1000 9c26 0000 0000 0000  .........&......
0003bfe0: 0000 9800 0300 1000 9b26 0000 0000 0000  .........&......
```

e09f 2000转换二进制取15位，转换十六进制后为1FE0，3a000 + 1FE0 = 3BFE0，也就是0000 9800 0300 1000 9b26 0000 0000 0000这行数据


```sql
test=# select * from bt_page_items('idx_2',29);

 itemoffset |   ctid   | itemlen | nulls | vars |          data           | dead |   htid   | tids 
------------+----------+---------+-------+------+-------------------------+------+----------+------
          1 | (152,3)  |      16 | f     | f    | 9b 26 00 00 00 00 00 00 | f    | (152,3)  | 
          2 | (152,4)  |      16 | f     | f    | 9c 26 00 00 00 00 00 00 | f    | (152,4)  | 
          3 | (152,5)  |      16 | f     | f    | 9d 26 00 00 00 00 00 00 | f    | (152,5)  | 
	...
        116 | (153,53) |      16 | f     | f    | 0e 27 00 00 00 00 00 00 | f    | (153,53) | 
        117 | (153,54) |      16 | f     | f    | 0f 27 00 00 00 00 00 00 | f    | (153,54) | 
        118 | (153,55) |      16 | f     | f    | 10 27 00 00 00 00 00 00 | f    | (153,55) | 
(118 rows)
```




## 二层结构

记录数超过1层结构的索引可以存储的记录数时，会分裂为2层结构，除了meta page和root page，还可能包含1层branch page以及1层leaf page。
如果是边界页(branch or leaf)，那么其中一个方向没有PAGE，这个方向的链表信息都统一指向meta page。

![image](https://github.com/user-attachments/assets/5180fe80-653e-4a32-bd25-9cf38e5c095e)


<img width="825" alt="image" src="https://github.com/user-attachments/assets/fdefadd3-c4fc-46f0-b10f-0ff4f783b18d" />



```sql
test=# select * from bt_metap('t_idx_3_pkey');
 magic  | version | root | level | fastroot | fastlevel | last_cleanup_num_delpages | last_cleanup_num_tuples | allequalimage 
--------+---------+------+-------+----------+-----------+---------------------------+-------------------------+---------------
 340322 |       4 |  420 |     2 |      420 |         2 |                         8 |                      -1 | t
(1 row)

(END)
```

```sql
test=# select * from bt_page_stats('t_idx_3_pkey', 420);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
   420 | r    |         10 |          0 |            15 |      8192 |      7956 |         0 |         0 |          2 |          2
(1 row)

-- 查看root page存储的 branch page items (指向branch page)
test=# select * from bt_page_items('t_idx_3_pkey', 420);
 itemoffset |   ctid   | itemlen | nulls | vars |          data           | dead | htid | tids 
------------+----------+---------+-------+------+-------------------------+------+------+------
          1 | (3,0)    |       8 | f     | f    |                         |      |      | 
          2 | (419,1)  |      16 | f     | f    | 77 97 01 00 00 00 00 00 |      |      | 
          3 | (706,1)  |      16 | f     | f    | ed 2e 03 00 00 00 00 00 |      |      | 
          4 | (992,1)  |      16 | f     | f    | 63 c6 04 00 00 00 00 00 |      |      | 
          5 | (1278,1) |      16 | f     | f    | d9 5d 06 00 00 00 00 00 |      |      | 
          6 | (1564,1) |      16 | f     | f    | 4f f5 07 00 00 00 00 00 |      |      | 
          7 | (1850,1) |      16 | f     | f    | c5 8c 09 00 00 00 00 00 |      |      | 
          8 | (2136,1) |      16 | f     | f    | 3b 24 0b 00 00 00 00 00 |      |      | 
          9 | (2422,1) |      16 | f     | f    | b1 bb 0c 00 00 00 00 00 |      |      | 
         10 | (2708,1) |      16 | f     | f    | 27 53 0e 00 00 00 00 00 |      |      | 
(10 rows)

-- 根据branch page id(ctid的块号，branch page层的ctid是指向自身的page)查看stats
-- btpo_level = 1说明不是最底层，btpo_flags = 0说明是branch page，btpo_next指向下一个branch page的id
test=# select * from bt_page_stats('t_idx_3_pkey', 3);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     3 | i    |        286 |          0 |            15 |      8192 |      2436 |         0 |       419 |          1 |          0
(1 row)

-- 查看第三页branch page的内容，data里同样存储了77 97 01 00 00 00 00 00，下一页branch page的最大值
test=# select * from bt_page_items('t_idx_3_pkey', 3);
 itemoffset |  ctid   | itemlen | nulls | vars |          data           | dead | htid | tids 
------------+---------+---------+-------+------+-------------------------+------+------+------
          1 | (295,1) |      16 | f     | f    | 77 97 01 00 00 00 00 00 |      |      | 
          2 | (2,0)   |       8 | f     | f    |                         |      |      | 
          3 | (11,1)  |      16 | f     | f    | 6f 01 00 00 00 00 00 00 |      |      | 

test=# select * from bt_page_stats('t_idx_3_pkey', 419);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
   419 | i    |        286 |          0 |            15 |      8192 |      2436 |         3 |       706 |          1 |          0
(1 row)

-- 查看branch page的内容，里面存储了 leaf page ctid (指向leaf page)
-- 只要不是最右边的页，第一条都代表右页的起始item。第二条才是当前页的起始ctid。注意所有branch page的起始item对应的data都是空的。
-- 也就是说它不存储当前branch page包含的所有leaf pages的索引字段内容的最小值。
test=# select * from bt_page_items('t_idx_3_pkey', 419);
 itemoffset |  ctid   | itemlen | nulls | vars |          data           | dead | htid | tids 
------------+---------+---------+-------+------+-------------------------+------+------+------
          1 | (582,1) |      16 | f     | f    | ed 2e 03 00 00 00 00 00 |      |      | 
          2 | (295,0) |       8 | f     | f    |                         |      |      | 
          3 | (296,1) |      16 | f     | f    | e5 98 01 00 00 00 00 00 |      |      | 
          4 | (297,1) |      16 | f     | f    | 53 9a 01 00 00 00 00 00 |      |      | 

-- 查看branch page所指向的leaf page的状态
-- btpo_level = 0说明是最底层，btpo_flags = 1说明是一个leaf page
test=# select * from bt_page_stats('t_idx_3_pkey', 582);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
   582 | l    |        367 |          0 |            16 |      8192 |       808 |       581 |       583 |          0 |          1
(1 row)








-- 特殊的页
-- btpo_flags = 261：page from block 5 is deleted
test=# select * from bt_page_stats('t_idx_3_pkey', 5);
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags 
-------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+------------
     5 | d    |          0 |          0 |             0 |      8192 |      8140 |         9 |         8 |          0 |        261
(1 row)

```


总的来说，2层结构和1层结构一样，中间多了一层branch page层，同样的：
1. branch page中记录了下一页的起始值
2. branch page中的ctid指向了下一层leaf page的块号
3. leaf page的ctid指向了表数据文件的块号和块内元组索引序号





## 多层结构

除了meta page，还可能包含多层branch page，以及一层leaf page。

![image](https://github.com/user-attachments/assets/fa0530cf-89f9-4d2a-870b-0cb7a2b2ce57)












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



ctid(block, offet):
![image](https://github.com/user-attachments/assets/7c542355-201a-4a2a-b17f-5d5729e29f21)





