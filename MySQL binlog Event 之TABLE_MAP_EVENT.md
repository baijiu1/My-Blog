# 包格式


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
