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


numeric_in()，函数入口，在src/backend/utils/adt/numeric.c，635行

如果判断为十进制的numric，那么走set_var_from_str函数

```c++
#define NUMERIC_SIGN_MASK	0xC000
#define NUMERIC_POS			0x0000
#define NUMERIC_NEG			0x4000
#define NUMERIC_SHORT		0x8000
#define NUMERIC_SPECIAL		0xC000
if (dweight >= 0)
    // 如果传入的值带小数点，weight算法是这么算的
    weight = (dweight + 1 + DEC_DIGITS - 1) / DEC_DIGITS - 1;
else
    weight = -((-dweight - 1) / DEC_DIGITS + 1);

offset = (weight + 1) * DEC_DIGITS - (dweight + 1); // 第一个元组前面补零的个数
ndigits = (ddigits + offset + DEC_DIGITS - 1) / DEC_DIGITS; // ddigits：跳过了第一个元组，剩余的位数。ndigits：数组元素的个数
sign是符号位，正数用NUMERIC_POS，负数用NUMERIC_NEG
dscale是小数部分的个数，代码部分：
if (isdigit((unsigned char) *cp))
{
	decdigits[i++] = *cp++ - '0';
	if (!have_dp)
		dweight++;
	else
		dscale++;
}

// n_header的计算方式
#define NUMERIC_SHORT_SIGN_MASK			0x2000
#define NUMERIC_SHORT_DSCALE_MASK		0x1F80
#define NUMERIC_SHORT_DSCALE_SHIFT		7
#define NUMERIC_SHORT_DSCALE_MAX		\
	(NUMERIC_SHORT_DSCALE_MASK >> NUMERIC_SHORT_DSCALE_SHIFT)
#define NUMERIC_SHORT_WEIGHT_SIGN_MASK	0x0040
#define NUMERIC_SHORT_WEIGHT_MASK		0x003F
#define NUMERIC_SHORT_WEIGHT_MAX		NUMERIC_SHORT_WEIGHT_MASK
#define NUMERIC_SHORT_WEIGHT_MIN		(-(NUMERIC_SHORT_WEIGHT_MASK+1))
#define SET_VARSIZE(PTR, len)				SET_VARSIZE_4B(PTR, len)
#define SET_VARSIZE_SHORT(PTR, len)			SET_VARSIZE_1B(PTR, len)
#define SET_VARSIZE_COMPRESSED(PTR, len)	SET_VARSIZE_4B_C(PTR, len)

#define SET_VARSIZE_4B(PTR,len) \
	(((varattrib_4b *) (PTR))->va_4byte.va_header = (((uint32) (len)) << 2))
#define SET_VARSIZE_4B_C(PTR,len) \
	(((varattrib_4b *) (PTR))->va_4byte.va_header = (((uint32) (len)) << 2) | 0x02)
#define SET_VARSIZE_1B(PTR,len) \
	(((varattrib_1b *) (PTR))->va_header = (((uint8) (len)) << 1) | 0x01)
#define SET_VARTAG_1B_E(PTR,tag) \
	(((varattrib_1b_e *) (PTR))->va_header = 0x01, \
	 ((varattrib_1b_e *) (PTR))->va_tag = (tag))

typedef union
{
	struct						/* Normal varlena (4-byte length) */
	{
		uint32		va_header;
		char		va_data[FLEXIBLE_ARRAY_MEMBER];
	}			va_4byte;
	struct						/* Compressed-in-line format */
	{
		uint32		va_header;
		uint32		va_tcinfo;	/* Original data size (excludes header) and
								 * compression method; see va_extinfo */
		char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Compressed data */
	}			va_compressed;
} varattrib_4b;

#define NUMERIC_CAN_BE_SHORT(scale,weight) \
	((scale) <= NUMERIC_SHORT_DSCALE_MAX && \
	(weight) <= NUMERIC_SHORT_WEIGHT_MAX && \
	(weight) >= NUMERIC_SHORT_WEIGHT_MIN)

// 等价替换： (-(0x003F+1)) = -64; 0x00FF = 255; 0x003F = 65;
( (scale) <= 0x00FF && (weight) <= 0x003F && (weight) >= -64 )
n = var->ndigits;
/* truncate leading zeroes */
while (n > 0 && *digits == 0)
{
	digits++;
	weight--;
	n--;
}
/* truncate trailing zeroes */
while (n > 0 && digits[n - 1] == 0)
	n--;

/* If zero result, force to weight=0 and positive sign */
if (n == 0)
{
	weight = 0;
	sign = NUMERIC_POS;
}
if (NUMERIC_CAN_BE_SHORT(var->dscale, weight))
{
	// 长度计算，整个numric的存储长度
	// #define VARHDRSZ ((int32) sizeof(int32))
	// #define NUMERIC_HDRSZ_SHORT (VARHDRSZ + sizeof(uint16))
	// NUMERIC_HDRSZ_SHORT： (((int32) sizeof(int32)) + sizeof(uint16))
	// n的值为ndigits
	// typedef int16 NumericDigit;
	// len = (((int32) sizeof(int32)) + sizeof(uint16)) + ndigits * sizeof(int16)
	len = NUMERIC_HDRSZ_SHORT + n * sizeof(NumericDigit);
	result = (Numeric) palloc(len);
	// SET_VARSIZE： ( ((varattrib_4b *) (PTR))->va_4byte.va_header = (((uint32) (len)) << 2))
	SET_VARSIZE(result, len);
	result->choice.n_short.n_header =
		// sign如果为正数则定义为NUMERIC_POS，负数定义为NUMERIC_ENG
		// 如果为正数，那sign为0x8000，如果为负数，那sign为(NUMERIC_SHORT | NUMERIC_SHORT_SIGN_MASK) = 0xA000，十进制为40960
		(sign == NUMERIC_NEG ? (NUMERIC_SHORT | NUMERIC_SHORT_SIGN_MASK)
		 : NUMERIC_SHORT)
		| (var->dscale << NUMERIC_SHORT_DSCALE_SHIFT)
		| (weight < 0 ? NUMERIC_SHORT_WEIGHT_SIGN_MASK : 0)
		| (weight & NUMERIC_SHORT_WEIGHT_MASK);
}
else
{
	len = NUMERIC_HDRSZ + n * sizeof(NumericDigit);
	result = (Numeric) palloc(len);
	SET_VARSIZE(result, len);
	result->choice.n_long.n_sign_dscale =
		sign | (var->dscale & NUMERIC_DSCALE_MASK);
	result->choice.n_long.n_weight = weight;
}

// 头部字节的计算方式
// 计算出来len之后SET_VARSIZE(result, len);计算出来vl_len_的值
#define VARATT_CONVERTED_SHORT_SIZE(PTR) \
	(VARSIZE(PTR) - VARHDRSZ + VARHDRSZ_SHORT)
#define VARHDRSZ_SHORT			offsetof(varattrib_1b, va_data)
#define VARSIZE(PTR) ((((varattrib_4b *) (PTR))->va_4byte.va_header >> 2) & 0x3FFFFFFF)
#define VARHDRSZ ((int32) sizeof(int32))
typedef struct
{
	uint8		va_header;
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Data begins here */
} varattrib_1b;
#define SET_VARSIZE_SHORT(PTR, len)			SET_VARSIZE_1B(PTR, len)
#define SET_VARSIZE_1B(PTR,len) \
	(((varattrib_1b *) (PTR))->va_header = (((uint8) (len)) << 1) | 0x01)

// 代码位于src/backend/access/common/heaptuple.c 352行
else if (VARLENA_ATT_IS_PACKABLE(att) &&
	VARATT_CAN_MAKE_SHORT(val))
{
	/* convert to short varlena -- no alignment */
	data_length = VARATT_CONVERTED_SHORT_SIZE(val);
	SET_VARSIZE_SHORT(data, data_length);
	memcpy(data + 1, VARDATA(val), data_length - 1);
}

// VARATT_CONVERTED_SHORT_SIZE的计算方式
((((varattrib_4b *) (PTR))->va_4byte.va_header >> 2) & 0x3FFFFFFF) - ((int32) sizeof(int32)) + offsetof(varattrib_1b, va_data) = 9
// (( (48) >> 2) & 0x3FFFFFFF) - ((int32) sizeof(int32)) + 1 = 9

// 得出头部字节的结果
SET_VARSIZE_SHORT(data, data_length);
(((varattrib_1b *) (data))->va_header = (((uint8) (9)) << 1) | 0x01) = 13
```

##### 举例：
插入一个小数：12323424.567，会得到以下结果：
```bash
dweight: 7  小数点前的位数，初始值为-1，所以会少一位
DEC_DIGITS: 4 宏定义，默认为4
weight: 1    表示的是整数部分所占用的数组元素个数
offset: 0   第一个元组要补零的个数。102323424.567中，offset值是3，因为按照4位元组存储，为：0001 0232 3424，前面要补3个0
ddigits: 11 有效位数，也就是12323424.567，总共11位
ndigits: 3 指的digits数组元素的个数
```
```bash
weight表示的是整数部分所占用的数组元素个数，不过进行了一系列的运算，在保证有整数部分， weight = （整数部分个数 + 4 - 1）/4 - 1
weight = (dweight + 1 + DEC_DIGITS - 1) / DEC_DIGITS - 1; // (7 + 1 + 4 - 1) / 4 - 1
( (3) <= 0x00FF && (1) <= 0x003F && (1) >= -64 )
```

下面是物理存储结构：

```bash
00001fd0: 0000 0000 0000 0000 0603 0000 0000 0000  ................
00001fe0: 0000 0000 0000 0000 0100 0100 0208 1800  ................
00001ff0: 1381 81d0 0460 0d26 1600 0000 0000 0000  .....`.&........
00002000: 0a  
```

解析物理文件内容过程：
将所有数字转换成16进制：

```bash
1232 3424 5670
04d0 0d60 1626
```

计算n_header：
```bash
0x8000 | (3 << 7) | (0) | (1 & 0x003F) = 0x8181
```

计算头一个字节的值：

```c++
#define VARATT_CONVERTED_SHORT_SIZE(PTR) \
	(VARSIZE(PTR) - VARHDRSZ + VARHDRSZ_SHORT)
#define VARHDRSZ_SHORT			offsetof(varattrib_1b, va_data)
#define VARSIZE(PTR) ((((varattrib_4b *) (PTR))->va_4byte.va_header >> 2) & 0x3FFFFFFF)
#define VARHDRSZ ((int32) sizeof(int32))
typedef struct
{
	uint8		va_header;
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Data begins here */
} varattrib_1b;
((((varattrib_4b *) (PTR))->va_4byte.va_header >> 2) & 0x3FFFFFFF) - ((int32) sizeof(int32)) + offsetof(varattrib_1b, va_data) = 9
val 48 len:9 data:13
(( (48) >> 2) & 0x3FFFFFFF) - ((int32) sizeof(int32)) + 1 = 9

SET_VARSIZE_SHORT(data, data_length);
(((varattrib_1b *) (data))->va_header = (((uint8) (9)) << 1) | 0x01) = 13
```

###### numeric类型还原的过程
函数位于numeric_out();这个函数就是还原numeric的过程。整体函数过程比较简单，调用了2个函数init_var_from_num()和get_str_from_var()。
分别是init_var_from_num负责解析ndigits，weight，dscale等等。get_str_from_var负责转换为string
PG_GETARG_NUMERIC：这是比较重要的一个部分，负责判断传入的数值是否为short类型，还是long类型，是否是压缩还是非压缩

```c++
Datum
numeric_out(PG_FUNCTION_ARGS)
{
	Numeric		num = PG_GETARG_NUMERIC(0);
	NumericVar	x;
	char	   *str;

	/*
	 * Handle NaN and infinities
	 */
	if (NUMERIC_IS_SPECIAL(num))
	{
		if (NUMERIC_IS_PINF(num))
			PG_RETURN_CSTRING(pstrdup("Infinity"));
		else if (NUMERIC_IS_NINF(num))
			PG_RETURN_CSTRING(pstrdup("-Infinity"));
		else
			PG_RETURN_CSTRING(pstrdup("NaN"));
	}

	/*
	 * Get the number in the variable format.
	 */
	init_var_from_num(num, &x);

	str = get_str_from_var(&x);

	PG_RETURN_CSTRING(str);
}
```

```c++
// numeric_out()函数调用了Numeric		num = PG_GETARG_NUMERIC(0);，以下是调用过程
#define PG_GETARG_NUMERIC(n)	  DatumGetNumeric(PG_GETARG_DATUM(n))
static inline Numeric
DatumGetNumeric(Datum X)
{
	// 返回一个Numeric类型的变量
	return (Numeric) PG_DETOAST_DATUM(X);
}

#define PG_DETOAST_DATUM(datum) \
	pg_detoast_datum((struct varlena *) DatumGetPointer(datum))
struct varlena *
pg_detoast_datum(struct varlena *datum)
{
	// 如果判断不是4B_U类型的话，执行detoast_attr函数。否则的话，就直接返回前4个字节，按小端序排序
	// VARATT_IS_EXTENDED: (!VARATT_IS_4B_U(PTR)) 
	if (VARATT_IS_EXTENDED(datum))
		return detoast_attr(datum);
	else
		return datum;
}

// 总体来说，这个函数就是判断传入的numeric是不是short还是long，是压缩还是非压缩，是外部存储还是直接存储。都是通过判断va_header来判断的，也就是物理文件的第一个字节
// 这里会计算好va_header的值
struct varlena *
detoast_attr(struct varlena *attr)
{
	if (VARATT_IS_EXTERNAL_ONDISK(attr))
	{
		/*
		 * This is an externally stored datum --- fetch it back from there
		 */
		attr = toast_fetch_datum(attr);
		/* If it's compressed, decompress it */
		if (VARATT_IS_COMPRESSED(attr))
		{
			struct varlena *tmp = attr;

			attr = toast_decompress_datum(tmp);
			pfree(tmp);
		}
	}
	else if (VARATT_IS_EXTERNAL_INDIRECT(attr))
	{
		/*
		 * This is an indirect pointer --- dereference it
		 */
		struct varatt_indirect redirect;

		VARATT_EXTERNAL_GET_POINTER(redirect, attr);
		attr = (struct varlena *) redirect.pointer;

		/* nested indirect Datums aren't allowed */
		Assert(!VARATT_IS_EXTERNAL_INDIRECT(attr));

		/* recurse in case value is still extended in some other way */
		attr = detoast_attr(attr);

		/* if it isn't, we'd better copy it */
		if (attr == (struct varlena *) redirect.pointer)
		{
			struct varlena *result;

			result = (struct varlena *) palloc(VARSIZE_ANY(attr));
			memcpy(result, attr, VARSIZE_ANY(attr));
			attr = result;
		}
	}
	else if (VARATT_IS_EXTERNAL_EXPANDED(attr))
	{
		/*
		 * This is an expanded-object pointer --- get flat format
		 */
		attr = detoast_external_attr(attr);
		/* flatteners are not allowed to produce compressed/short output */
		Assert(!VARATT_IS_EXTENDED(attr));
	}
	else if (VARATT_IS_COMPRESSED(attr))
	{
		/*
		 * This is a compressed value inside of the main tuple
		 */
		attr = toast_decompress_datum(attr);
	}
	else if (VARATT_IS_SHORT(attr))
	{
		/*
		 * This is a short-header varlena --- convert to 4-byte header format
		 */
		// 如果是short类型，这里通过物理存储的第一个字节来计算va_header的值，后面计算ndigits的值会用到
		// size data_size = ((((varattrib_1b *) (attr))->va_header >> 1) & 0x7F) - __builtin_offsetof(varattrib_1b, va_data)
		// __builtin_offsetof(varattrib_1b, va_data)取的是va_data的偏移量，因为varattrib_1b的header恒定是uint8_t，所以就是一个字节，值也就是1
		Size		data_size = VARSIZE_SHORT(attr) - VARHDRSZ_SHORT;
		// size new_size = data_size + ((int32) sizeof(int32))
		// ((int32) sizeof(int32))因为是int32，所以就是4个字节
		Size		new_size = data_size + VARHDRSZ;
		struct varlena *new_attr;

		new_attr = (struct varlena *) palloc(new_size);
		// 这里计算va_header的值，扩展为：(((varattrib_4b *) (new_attr))->va_4byte.va_header = (((uint32) (new_size)) << 2))
		SET_VARSIZE(new_attr, new_size);
		memcpy(VARDATA(new_attr), VARDATA_SHORT(attr), data_size);
		attr = new_attr;
	}

	return attr;
}
```

这里会计算numeric的各个部分，如ndigits、weight、sign等等

```c++
static void
init_var_from_num(Numeric num, NumericVar *dest)
{
	dest->ndigits = NUMERIC_NDIGITS(num);
	dest->weight = NUMERIC_WEIGHT(num);
	dest->sign = NUMERIC_SIGN(num);
	dest->dscale = NUMERIC_DSCALE(num);
	dest->digits = NUMERIC_DIGITS(num);
	dest->buf = NULL;			/* digits array is not palloc'd */
}

具体计算方式为：
ndigits：
((((((varattrib_4b *) (num))->va_4byte.va_header >> 2) & 0x3FFFFFFF) - (((int32) sizeof(int32)) + sizeof(uint16) + ((((num)->choice.n_header & 0x8000) != 0) ? 0 : sizeof(int16)))) / sizeof(NumericDigit))
va_header就是上面计算得出的数值，代入进去就可以计算出ndigits的值，也就是数组元素的个数

weight：
(((((num))->choice.n_header & 0x8000) != 0) ? (((num)->choice.n_short.n_header & 0x0040 ? ~0x003F : 0) | ((num)->choice.n_short.n_header & 0x003F)) : ((num)->choice.n_long.n_weight))
n_header是标志位，对于short格式来说，就是第二个和第三个字节的内容，转换小端序后代入到上面公式里，下面的n_header都是一样的

sign：
((((num)->choice.n_header & 0xC000) == 0x8000) ? (((num)->choice.n_short.n_header & 0x2000) ? 0x4000 : 0x0000) : ((((num)->choice.n_header & 0xC000) == 0xC000) ? ((num)->choice.n_header & 0xF000) : ((num)->choice.n_header & 0xC000)))

dscale：
(((((num))->choice.n_header & 0x8000) != 0) ? ((num)->choice.n_short.n_header & 0x1F80) >> 7 : ((num)->choice.n_long.n_sign_dscale & 0x3FFF))

digits：
((((num)->choice.n_header & 0x8000) != 0) ? (num)->choice.n_short.n_data : (num)->choice.n_long.n_data)

```




### float4

pg中是按照标准 IEEE 754 浮点数格式来存储的。

转换方式：
1、先将给出的精度数据转换为IEEE 754 精确表示：

```c++
double value = 19425.239482;
std::cout << std::setprecision(50) << value << std::endl;
```
2、将计算出来的数值转换为二进制
3、根据SEM分别计算，最后转换为16进制


     现在让我们按照IEEE浮点数表示法，一步步的将float型浮点数12345转换为十六进制代码。首先数字是正整数，所以符号位为0，接下来12345的二进制表示为11000000111001，小数点向左移，一直移到离最高位只有1位，就是最高位的1。即1.1000000111001*2^13，所有的二进制数字最前边都有一个1，所以可以去掉，那么尾数位的精确度其实可以为24 bit。再来看指数位，因为是有8 bit，所以只为能够表示的为0~255，也可以说是-128~127，所以指数为为正的话，必须加上127，即13+127=140，即10001100。好了，所有的数据都整理好了，现在表示12345的float存储方式即01000110010000001110010000000000，现在把它转化为16进制，即4640 e400，而存储文件是从下向上写入的，所以表示为 e400 4640。

代码位于：arc/backend/utils/adt/float.c 函数入口：float4in

```c++
#define PG_RETURN_FLOAT4(x)  return Float4GetDatum(x)
static inline Datum
Float4GetDatum(float4 X)
{
	union
	{
		float4		value;
		int32		retval;
	}			myunion;

	myunion.value = X;
	return Int32GetDatum(myunion.retval);
}

```


### double
pg中是按照标准 IEEE 754 浮点数格式来存储的。

转换方式：
1、先将给出的精度数据转换为IEEE 754 精确表示：

```c++
double value = 19425.239482;
std::cout << std::setprecision(15) << value << std::endl;
```
2、将计算出来的数值转换为二进制
3、根据SEM分别计算，最后转换为16进制
```c++
#define PG_RETURN_FLOAT8(x)  return Float8GetDatum(x)

```


# 变长类型
### char、char(n) 、character(n)、bpchar、bpchar(n)
这些（这些类型都是bpchar的马甲）是同一种类型，使用的是同一个输入输出函数。 
### character(n) 、varchar、varchar(n)、character varying(n)
这些（这些类型都是varchar的马甲）是同一种类型，使用的是相同的输入输出函数。

```c++
#define VARATT_IS_4B(PTR) \
	((((varattrib_1b *) (PTR))->va_header & 0x01) == 0x00)
#define VARATT_IS_4B_U(PTR) \
	((((varattrib_1b *) (PTR))->va_header & 0x03) == 0x00)
#define VARATT_IS_4B_C(PTR) \
	((((varattrib_1b *) (PTR))->va_header & 0x03) == 0x02)
#define VARATT_IS_1B(PTR) \
	((((varattrib_1b *) (PTR))->va_header & 0x01) == 0x01)
#define VARATT_IS_1B_E(PTR) \
	((((varattrib_1b *) (PTR))->va_header) == 0x01)
#define VARATT_NOT_PAD_BYTE(PTR) \
	(*((uint8 *) (PTR)) != 0)

/* VARSIZE_4B() should only be used on known-aligned data */
#define VARSIZE_4B(PTR) \
	((((varattrib_4b *) (PTR))->va_4byte.va_header >> 2) & 0x3FFFFFFF)
#define VARSIZE_1B(PTR) \
	((((varattrib_1b *) (PTR))->va_header >> 1) & 0x7F)
#define VARTAG_1B_E(PTR) \
	(((varattrib_1b_e *) (PTR))->va_tag)

#define SET_VARSIZE_4B(PTR,len) \
	(((varattrib_4b *) (PTR))->va_4byte.va_header = (((uint32) (len)) << 2))
#define SET_VARSIZE_4B_C(PTR,len) \
	(((varattrib_4b *) (PTR))->va_4byte.va_header = (((uint32) (len)) << 2) | 0x02)
#define SET_VARSIZE_1B(PTR,len) \
	(((varattrib_1b *) (PTR))->va_header = (((uint8) (len)) << 1) | 0x01)
#define SET_VARTAG_1B_E(PTR,tag) \
	(((varattrib_1b_e *) (PTR))->va_header = 0x01, \
	 ((varattrib_1b_e *) (PTR))->va_tag = (tag))

typedef union
{
	struct						/* Normal varlena (4-byte length) */
	{
		uint32		va_header;
		char		va_data[FLEXIBLE_ARRAY_MEMBER];
	}			va_4byte;
	struct						/* Compressed-in-line format */
	{
		uint32		va_header;
		uint32		va_tcinfo;	/* Original data size (excludes header) and
								 * compression method; see va_extinfo */
		char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Compressed data */
	}			va_compressed;
} varattrib_4b;

typedef struct
{
	uint8		va_header;
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Data begins here */
} varattrib_1b;

/* TOAST pointers are a subset of varattrib_1b with an identifying tag byte */
typedef struct
{
	uint8		va_header;		/* Always 0x80 or 0x01 */
	uint8		va_tag;			/* Type of datum */
	char		va_data[FLEXIBLE_ARRAY_MEMBER]; /* Type-specific data */
} varattrib_1b_e;
```


注意这是通用格式，varlena还分为很多种格式，每种格式的定义都不相同。我们在使用之前，需要根据它的第一个字节，转换为它对应格式：


| tag | length |
|--------|-----|
 | 1bit  |   7bit|


第一个字节等于1000 0000, 那么就是varattrib_1b_e，用来存储 external 数据（在 toast 会有讲解到）
第一个字节的最高位等于1，然后字节不等于1000 0000，那么就是varattrib_1b，用来存储小数据，varattrib_1b类型只是用于存储长度不超过127 byte 的小数据
第一个字节的最高位等于0，那么就是varattrib_4b，可以存储不超过1GB的数据

##### 示例
插入一条数据

```bash
00001fc0: 0000 0000 0000 0000 1300 0000 0000 0000  ................
00001fd0: 0000 0000 0000 0000 0100 0200 0209 1800  ................
00001fe0: 0100 0000 2f56 6172 6961 626c 652d 6c65  ..../Variable-le
00001ff0: 6e67 7468 2073 7472 696e 6700 0000 0000  ngth string.....
```

从2f开始，把2f转换成二进制为：101111
最低一位标识类型，这里是varattrib_1b，高7位是长度，那么就是10111，转换十进制是23，也就是总长度23:

```bash
2f56 6172 6961 626c 652d 6c65 6e67 7468 2073 7472 696e 67
```

正好对应到了字符串


### name类型：

```c++
typedef struct nameData
{
	char		data[NAMEDATALEN];
} NameData;
typedef NameData *Name;
#define NAMEDATALEN 64;
```

```bash
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
```

liu转换为16进制:6c69 75

# 日期类型
<img width="1001" alt="image" src="https://github.com/user-attachments/assets/c14d817f-69c8-4904-93ea-e374b76a1652">

### date
typedef int32 DateADT;
PostgreSQL按照儒略日(Julian day,JD)，即公元前4713年1月1日作为起始
下面是计算方法：

```c++
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
```

比如：2012-12-08，通过date2j，最后计算得出4725，转换成16进制就是1275，到文件存储里就是：

```bash
00001fe0: 0300 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 7512 0000 0000 0000  ........u.......
00002000: 0a                                       .
```


### time & time with time zone

```c++
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
```

按照以下计算方式来计算最终存储结果：

```c++
#define USECS_PER_SEC	INT64CONST(1000000)
#define MINS_PER_HOUR	60
#define SECS_PER_MINUTE 60

result->time = ((tm->tm_hour * MINS_PER_HOUR + tm->tm_min) * SECS_PER_MINUTE) + tm->tm_sec + fsec;
```

举例，插入time类型：12:13:14.048476，文件中的格式为：

```bash
00001fe0: f802 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 dce7 3f3e 0a00 0000  ..........?>....
00002000: 0a
```

计算：((((12 * 60 + 13) * 60) + 14) * 1000000LL ) + 48476 = 1044375516;
转换16进制为：3e3f e7dc，转换成大端序：dce7 3f3e

timez类型：

依然也是按照上述计算方式，最后计算出的结果是54977187384，转换16进制就是：0ccce54e38，看下文件里怎么存储的。+8表示为：0xFFFF8F80：

```bash
00001fd0: 0000 0000 0000 0000 fa02 0000 0000 0000  ................
00001fe0: 0000 0000 0000 0000 0100 0100 0009 1800  ................
00001ff0: 384e e5cc 0c00 0000 808f ffff 0000 0000  8N..............
00002000: 0a                                       .
```

### timestamp 和 timestamp with time zone
源码位于：src/backend/utils/adt/timestamp.c

先看下定义：

```c++
typedef int64 Timestamp;
typedef int64 TimestampTz;
```

入口函数： timestamp_in

```c++
date = date2j(tm->tm_year, tm->tm_mon, tm->tm_mday) - POSTGRES_EPOCH_JDATE;
time = time2t(tm->tm_hour, tm->tm_min, tm->tm_sec, fsec);

*result = date * USECS_PER_DAY + time;
```

##### timestamp

```sql
test=# create table timesandtimestz(t1 timestamp(6));
CREATE TABLE
test=# insert into timesandtimestz values ('2024-12-01 15:38:25.192837');
INSERT 0 1
```

```bash
00001fe0: fc02 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 85ef ccfd 35cb 0200  ............5...
00002000: 0a                                       .
```

根据上面公式，计算出结果为：786382705192837，转换为： 2cb35fdccef85

##### timestamptz

```sql
test=# create table timesandtimestzz(t2 timestamptz(6));
CREATE TABLE
test=# insert into timesandtimestzz values ('2024-12-01 15:38:25.192837 +8:00:00');
INSERT 0 1
```

```bash
00001fe0: ff02 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 85cf 2f49 2fcb 0200  ........../I/...
00002000: 0a                                       .
```
带时区的话，会多出一个计算公式：

```c++
*result = dt2local(*result, -(*tzp)); // 参数里的result就是timestamp计算出的值 tzp就是时区
static Timestamp
dt2local(Timestamp dt, int timezone)
{
	dt -= (timezone * USECS_PER_SEC);
	return dt;
}
```

+8:00:00转换秒，会被计算为：-28800，最后根据上面dt2local函数计算出来的结果为：786353905192837，转换16进制就是：2cb2f492fcf85


##### interval


后续再研究



### 对象标识符类型 

oid类型

```sql
test=# create table oidt(o1 oid);
CREATE TABLE
test=# insert into oidt values(100);
INSERT 0 1
```

```c++
typedef unsigned int Oid;
```

```bash
00001fe0: 0103 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 6400 0000 0000 0000  ........d.......
00002000: 0a                                       .
```
100转换16进制为： 64，占四个字节


### 布尔型 
bool：基础类型，占位1字节。以0、1来表示false, true。 

```sql
test=# create table boolt(b1 bool);
CREATE TABLE
test=# insert into boolt values ('1');
INSERT 0 1
test=# insert into boolt values ('0');
INSERT 0 1
```

```bash
00001fc0: 0403 0000 0000 0000 0000 0000 0000 0000  ................
00001fd0: 0200 0100 0008 1800 0000 0000 0000 0000  ................
00001fe0: 0303 0000 0000 0000 0000 0000 0000 0000  ................
00001ff0: 0100 0100 0008 1800 0100 0000 0000 0000  ................
00002000: 0a                                       .                                     .
```
01为true，00为false，由于pg是默认8字节对齐，所以后面补齐了7字节

### 二进制类型 

```c++
typedef struct varlena bytea;
```

实际上也是varlena类型，这部分在《变长类型》部分有讲解






















