# 数值类型
### tinyint
2个字节，最大到255，也就是ff

### smallint
2个字节，最大到65535，有符号32767，也就是最大ffff

### integer
占4个字节，不够用00填充

### bigint
占8个字节，不够用00填充


### numeric

占位，现在还没弄懂

### float4

     现在让我们按照IEEE浮点数表示法，一步步的将float型浮点数12345转换为十六进制代码。首先数字是正整数，所以符号位为0，接下来12345的二进制表示为11000000111001，小数点向左移，一直移到离最高位只有1位，就是最高位的1。即1.1000000111001*2^13，所有的二进制数字最前边都有一个1，所以可以去掉，那么尾数位的精确度其实可以为24 bit。再来看指数位，因为是有8 bit，所以只为能够表示的为0~255，也可以说是-128~127，所以指数为为正的话，必须加上127，即13+127=140，即10001100。好了，所有的数据都整理好了，现在表示12345的float存储方式即01000110010000001110010000000000，现在把它转化为16进制，即4640 e400，而存储文件是从下向上写入的，所以表示为 e400 4640。
     整数部分弄懂了，小数部分先跳过
### double



# 变长类型
### char、char(n) 、character(n)、bpchar、bpchar(n)
这些（这些类型都是bpchar的马甲）是同一种类型，使用的是同一个输入输出函数。 
### character(n) 、varchar、varchar(n)、character varying(n)
这些（这些类型都是varchar的马甲）是同一种类型，使用的是相同的输入输出函数。
  
struct varlena
{
	char		vl_len_[4];		/* Do not touch this field directly! */
	char		vl_dat[FLEXIBLE_ARRAY_MEMBER];	/* Data content is here */
};



注意这是通用格式，varlena还分为很多种格式，每种格式的定义都不相同。我们在使用之前，需要根据它的第一个字节，转换为它对应格式：

第一个字节等于1000 0000, 那么就是varattrib_1b_e，用来存储 external 数据（在 toast 会有讲解到）
第一个字节的最高位等于1，然后字节不等于1000 0000，那么就是varattrib_1b，用来存储小数据
第一个字节的最高位等于0，那么就是varattrib_4b，可以存储不超过1GB的数据

00001fc0: 0000 0000 0000 0000 1300 0000 0000 0000  ................
00001fd0: 0000 0000 0000 0000 0100 0200 0209 1800  ................
00001fe0: 0100 0000 2f56 6172 6961 626c 652d 6c65  ..../Variable-le
00001ff0: 6e67 7468 2073 7472 696e 6700 0000 0000  ngth string.....


从2f开始，把2f转换成二进制为：101111
最低一位标识类型，这里是varattrib_1b，高7位是长度，那么就是10111，转换十进制是23，也就是总长度23:
2f56 6172 6961 626c 652d 6c65 6e67 7468 2073 7472 696e 67
正好对应到了字符串


### name类型：
typedef struct nameData
{
	char		data[NAMEDATALEN];
} NameData;
typedef NameData *Name;
#define NAMEDATALEN 64;


00001f60: 0000 0000 0000 0000 0300 0000 0000 0000  ................
00001f70: 0000 0000 0000 0000 0100 0100 0008 1800  ................
00001f80: 6c69 7500 0000 0000 0000 0000 0000 0000  liu.............
00001f90: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001fa0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001fb0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001fc0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001fd0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001fe0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00002000: 0a                 .



liu:6c69 75

# 日期类型
<img width="1001" alt="image" src="https://github.com/user-attachments/assets/c14d817f-69c8-4904-93ea-e374b76a1652">

### date
typedef int32 DateADT;
PostgreSQL按照儒略日(Julian day,JD)，即公元前4713年1月1日作为起始
下面是计算方法：


int date2j(int y, int m, int d)
{
    int julian;
    int century;

    if (m > 2) {
        m += 1;
        y += 4800;
    } else {
        m += 13;
        y += 4799;
    }

    century = y / 100;
    julian = y * 365 - 32167;
    julian += y / 4 - century + century / 4;
    julian += 7834 * m / 256 + d;

    return julian  - POSTGRES_EPOCH_JDATE;
} /* date2j() */

#define POSTGRES_EPOCH_JDATE 2451545 /* == date2j(2000, 1, 1) */

比如：2012-12-08，通过date2j，最后计算得出4725，转换成16进制就是1275，到文件存储里就是：

00001fe0: 0300 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 7512 0000 0000 0000  ........u.......
00002000: 0a                                       .


### time & time with time zone
#ifdef HAVE_INT64_TIMESTAMP
typedef int64 TimeADT;
#else
typedef float8 TimeADT;
#endif

typedef struct
{
	TimeADT		time;			/* all time units other than months and years */
	int32		zone;			/* numeric time zone, in seconds */
} TimeTzADT;

最终格式：

result->time = ((tm->tm_hour * MINS_PER_HOUR + tm->tm_min) * SECS_PER_MINUTE) + tm->tm_sec + fsec;






















