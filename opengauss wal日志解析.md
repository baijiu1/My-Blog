
```c++
typedef struct XLogPageHeaderData
{
	uint16		xlp_magic;		/* magic value for correctness checks */
	uint16		xlp_info;		/* flag bits, see below */
	TimeLineID	xlp_tli;		/* TimeLineID of first record on page */
	XLogRecPtr	xlp_pageaddr;	/* XLOG address of this page */

	/*
	 * When there is not enough space on current page for whole record, we
	 * continue on the next page.  xlp_rem_len is the number of bytes
	 * remaining from a previous page; it tracks xl_tot_len in the initial
	 * header.  Note that the continuation data isn't necessarily aligned.
	 */
	uint32		xlp_rem_len;	/* total len of remaining data for record */
} XLogPageHeaderData;

typedef struct XLogLongPageHeaderData
{
	XLogPageHeaderData std;		/* standard header fields */
	uint64		xlp_sysid;		/* system identifier from pg_control */
	uint32		xlp_seg_size;	/* just as a cross-check */
	uint32		xlp_xlog_blcksz;	/* just as a cross-check */
} XLogLongPageHeaderData;

typedef struct XLogRecord
{
	uint32		xl_tot_len;		/* total len of entire record */
	TransactionId xl_xid;		/* xact id */
	XLogRecPtr	xl_prev;		/* ptr to previous record in log */
	uint8		xl_info;		/* flag bits, see below */
	RmgrId		xl_rmid;		/* resource manager for this record */
	/* 2 bytes of padding here, initialize to zero */
	pg_crc32c	xl_crc;			/* CRC for this record */

	/* XLogRecordBlockHeaders and XLogRecordDataHeader follow, no padding */

} XLogRecord;

/*
 * Header info for block data appended to an XLOG record.日志记录中块数据的头信息
 */
typedef struct XLogRecordBlockHeader
{
	uint8		id;				/* block reference ID，块引用id */
	uint8		fork_flags;		/* fork within the relation, and flags，表中的分支以及标记位 */
	uint16		data_length;	/* number of payload bytes (not including page image)，荷载字节数，不包括页面镜像和XLogRecordBlockHeader结构体本身 */
 
	/* If BKPBLOCK_HAS_IMAGE, an XLogRecordBlockImageHeader struct follows，如果设置了BKPBLOCK_HAS_IMAGE，还会包含XLogRecordBlockImageHeader结构体 */
	/* If BKPBLOCK_SAME_REL is not set, a RelFileNode follows，如果没有设置BKPBLOCK_SAME_REL，则会包含RelFileNode */
	/* BlockNumber follows，后续为块号 */
} XLogRecordBlockHeader;
 
#define SizeOfXLogRecordBlockHeader (offsetof(XLogRecordBlockHeader, data_length) + sizeof(uint16))

typedef uint32 BlockNumber;
#define InvalidBlockNumber		((BlockNumber) 0xFFFFFFFF)
#define MaxBlockNumber			((BlockNumber) 0xFFFFFFFE)


/*
 * Additional header information when a full-page image is included
 * (i.e. when BKPBLOCK_HAS_IMAGE is set). 当包含full-page image（备份区块，即设置了BKPBLOCK_HAS_IMAGE）时，附加的头信息
 *
 * XLOG代码知道PG数据页通常在中间包含一些未使用的 hole(孔、洞，即空闲空间)，大小为零字节。既然我们知道hole都是零，因此可以从存储的数据中删除它(而且它也没有被计入XLOG记录的CRC中)。 因此，实际的块数据量为 BLCKSZ - hole的大小。
 *
* 另外，在启用wal_compression时，会在去掉hole后，尝试使用PGLZ压缩算法压缩full page image。这可以减小WAL容量，但会增加额外的CPU消耗。
 * 在这种情况下，由于hole的长度不能通过从BLCKSZ中减去page image字节数来计算，所以它基本上需要作为额外的信息来存储。但如果hole不存在，我们可以假设hole的大小为0，不需要存储额外的信息。
 * 请注意，如果压缩节省的字节数小于额外信息的长度，那么在WAL中存储page image的原始版本，而不是压缩后的版本。
 * 因此，当page image被成功压缩时，实际的块数据量小于BLCKSZ-hole的大小-额外信息的大小。
 */
 
typedef struct XLogRecordBlockImageHeader
{
	uint16		length;			/* number of page image bytes，页面镜像字节数 */
	uint16		hole_offset;	/* number of bytes before "hole"，hole前面的字节数 */
	uint8		bimg_info;		/* flag bits, see below，标记位 */
 
	/*
	 * If BKPIMAGE_HAS_HOLE and BKPIMAGE_IS_COMPRESSED, an
	 * XLogRecordBlockCompressHeader struct follows.
	 */
} XLogRecordBlockImageHeader;
 
/* Information stored in bimg_info */
#define BKPIMAGE_HAS_HOLE		0x01	     /* page image has "hole" */
#define BKPIMAGE_IS_COMPRESSED		0x02	 /* page image is compressed */
#define BKPIMAGE_APPLY		0x04 	/* page image should be restored during replay */


/*
 * Extra header information used when page image has "hole" and
 * is compressed.
 */
typedef struct XLogRecordBlockCompressHeader
{
	uint16		hole_length;	/* number of bytes in "hole" */
} XLogRecordBlockCompressHeader;
 
#define SizeOfXLogRecordBlockCompressHeader \
	sizeof(XLogRecordBlockCompressHeader)


typedef struct RelFileNode
{
	Oid			spcNode;		/* tablespace */
	Oid			dbNode;			/* database */
	Oid			relNode;		/* relation */
} RelFileNode;

/*
 * Maximum size of the header for a block reference. This is used to size a
 * temporary buffer for constructing the header. 
*/
#define MaxSizeOfXLogRecordBlockHeader \
	(SizeOfXLogRecordBlockHeader + \
	 SizeOfXLogRecordBlockImageHeader + \
	 SizeOfXLogRecordBlockCompressHeader + \
	 sizeof(RelFileNode) + \
	 sizeof(BlockNumber))

/*
 * These structs are currently not used in the code, they are here just for
 * documentation purposes. 这些结构体现在已经没有在代码中使用了，还保留在这里只是为了文档记录的目的。
 */
typedef struct XLogRecordDataHeaderShort
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_SHORT */
	uint8		data_length;	/* number of payload bytes */
}			XLogRecordDataHeaderShort;
 
#define SizeOfXLogRecordDataHeaderShort (sizeof(uint8) * 2)

typedef struct XLogRecordDataHeaderLong
{
	uint8		id;				/* XLR_BLOCK_ID_DATA_LONG */
	/* followed by uint32 data_length, unaligned */
}			XLogRecordDataHeaderLong;
 
#define SizeOfXLogRecordDataHeaderLong (sizeof(uint8) + sizeof(uint32))

XLOG Record按存储的数据内容来划分,大体可以分为三类:

Record for backup block（备份区块）：存储full-write-page的block，是为了解决日志页部分写的问题;
Record for tuple data block（非备份区块）：在full-write-page后，记录相应的page中的tuple变更
Record for Checkpoint：checkpoint发生时，在事务日志文件中记录checkpoint信息(其中包括Redo point).

```
![image](https://github.com/user-attachments/assets/53815389-2962-4784-a9be-fcd97fcbafe6)

