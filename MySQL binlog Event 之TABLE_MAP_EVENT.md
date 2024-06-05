# 前言
对于每个Write_rows_log_event、Update_rows_log_event、Delete_rows_log_event。都会在以上每一种event之前，写入一条Table_map_event来记录表相关的信息。

## Table_map_event
TABLE_MAP_EVENT记录了表的id，数据库名，表名，列长度，元数据等重要信息。这些信息在解析Write_rows_log_event、Update_rows_log_event、Delete_rows_log_event这些event的时候，有些字段会用到这些信息。
TABLE_MAP_EVENT只有在binlog_format=ROW格式时才会被记录。
TABLE_MAP_EVENT的设计目的，是为了当主库和从库之间有不同表定义的时候，复制仍能正常进行。
如果在一个事物中操作了多张表，多行记录，那么在binlog记录中，会对每行操作的event之前，都会记录一条对应表的TABLE_MAP_EVENT。

## 格式
对于每一个evnet来说，都具有相同且固定的header部分，如下：

| packet length  |        
|---------------|
| packet number  |
| response code  |
| timestamp  |
| event type |
------
table id   6字节

flags     2字节

schema name length   1字节

schema name    n+1字节   string<len>

table name length    1字节

table name     m+1字节  string<len>

field count    1或3或4字节
字段数量，占用1,3,4字节不等，根据具体数量决定


field types      field count 个字节
字段类型，每个类型占用1个字节

metadata
长度不定
表字段元数据，长度不定，也可能没有，由具体的表字段类型决定，通常不定长字段和浮点型字段都会有metadata


field is null
占用字节数为：(field_count + 7) / 8
表示表中哪些字段可以为空，每个字段使用1个bit表示
