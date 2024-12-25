
## base/16384解析

如何从base/16384找到pg_class和pg_attribute表的oid和filenode？

找到base/16384/pg_filenode.map文件

里面记录了表的oid对应到表的filenode，4字节(oid)：4字节(filenode)

pg_class的oid固定为1259，十六进制为04eb

pg_attribute的oid固定为1249，十六进制为04e1

从而找到filenode
