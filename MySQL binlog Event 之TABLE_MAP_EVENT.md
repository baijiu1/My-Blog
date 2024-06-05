# 前言
对于每个Write_rows_log_event、Update_rows_log_event、Delete_rows_log_event。都会在以上每一种event之前，写入一条Table_map_event来记录表相关的信息。

## Table_map_event
TABLE_MAP_EVENT记录了表的id，数据库名，表名，列长度，元数据等重要信息。这些信息在解析Write_rows_log_event、Update_rows_log_event、Delete_rows_log_event这些event的时候，有些字段会用到这些信息。
TABLE_MAP_EVENT只有在binlog_format=ROW格式时才会被记录。
TABLE_MAP_EVENT的设计目的，是为了当主库和从库之间有不同表定义的时候，复制仍能正常进行。
如果在一个事物中操作了多张表，多行记录，那么在binlog记录中，会对每行操作的event之前，都会记录一条对应表的TABLE_MAP_EVENT。

## 格式
对于每一个evnet来说，都具有相同且固定的header部分，如下：

| 字段 |    字节数    |  说明 |
|---------|--------|---------|
| packet length  |   3  | 包的总长度 |
| packet number  |  1   | 包的序列号，总是从0开始递增 |
| response code  |  1  | 标记该包是否成功 |
| timestamp  |     4   | event产生的时间 |
| event type |     1 |  event的类型  |
| server ID   |  4   |    |
| event size |   4 |  event大小  |
| binlog positon |  4 | binlog的位点 |
| binlog flags |  2 |  binlog的标志位  |
| body     |     |  对应到下面的结构  |
| check sum |  4 |  每一条event都有4个字节的check sum  |
--------

*body* 部分对应到以下结构：
|字段       | 字节数    | 说明 | 
|---------|--------|---------|
| table id            | 6  | 小端序，描述该表的table id，全局唯一 |
| flags               |  2 |   |
| schema name length    | 1   |  标识了database name的长度  |
| schema name           |   schema name length个字节   |   字节内容即为database name    |
| table name length    | 1   | 标识了table name的长度|
| table name            |   table name length个字节   |   字节内容即为table name   |
| field count        |   1或3或4字节不等   |   标识了列的长度，即有多少列   |
| field types        |  field count个字节  |    标识了当前表中的每个列的类型  |
| metadata length     |  根据field types记录的元数据，长度不等    |  标识了每个字段的元数据，有些列如long(int)没有元数据，有些列如char有元数据    |
| field is null         |   ((field_count + 7) / 8)个字节   |   标识列表中哪些字段可以为NULL，每个字段使用1个bit表示   |
--------







