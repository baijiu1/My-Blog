# 前言

在PG中，numeric类型一般用于要求精度极高的领域，今天来解析一下numeric类型在pg内部是如何解析，以及存储，存储后如何读取，反解析回数字。
在正式说明之前，首先我们需要看看在PG中的关于numeric的几个宏定义，这几个宏定义确定了numeric的存储类型，类型有1B、1B_E、4B、4B_U、4B_C

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
```

可以看到上面几个宏定义判断PTR指针下va_header和 0x01与的结果

# numeric_in()

numeric_in()函数是存储的入口函数，也就是说从前端拿到传入的小数，经过该函数的处理后，存储到了磁盘上。
直接上代码：

```c++
/*
 * numeric_in() -
 *
 *	Input function for numeric data type
 */
Datum
numeric_in(PG_FUNCTION_ARGS)
{
	char	   *str = PG_GETARG_CSTRING(0);
#ifdef NOT_USED
	Oid			typelem = PG_GETARG_OID(1);
#endif
	int32		typmod = PG_GETARG_INT32(2);
	Node	   *escontext = fcinfo->context;
	Numeric		res;
	const char *cp;
	const char *numstart;
	int			sign;

	/* Skip leading spaces */
	cp = str;
	while (*cp)
	{
		if (!isspace((unsigned char) *cp))
			break;
		cp++;
	}

	/*
	 * Process the number's sign. This duplicates logic in set_var_from_str(),
	 * but it's worth doing here, since it simplifies the handling of
	 * infinities and non-decimal integers.
	 */
	numstart = cp;
	sign = NUMERIC_POS;

	if (*cp == '+')
		cp++;
	else if (*cp == '-')
	{
		sign = NUMERIC_NEG;
		cp++;
	}

	/*
	 * Check for NaN and infinities.  We recognize the same strings allowed by
	 * float8in().
	 *
	 * Since all other legal inputs have a digit or a decimal point after the
	 * sign, we need only check for NaN/infinity if that's not the case.
	 */
	if (!isdigit((unsigned char) *cp) && *cp != '.')
	{
		/*
		 * The number must be NaN or infinity; anything else can only be a
		 * syntax error. Note that NaN mustn't have a sign.
		 */
		if (pg_strncasecmp(numstart, "NaN", 3) == 0)
		{
			res = make_result(&const_nan);
			cp = numstart + 3;
		}
		else if (pg_strncasecmp(cp, "Infinity", 8) == 0)
		{
			res = make_result(sign == NUMERIC_POS ? &const_pinf : &const_ninf);
			cp += 8;
		}
		else if (pg_strncasecmp(cp, "inf", 3) == 0)
		{
			res = make_result(sign == NUMERIC_POS ? &const_pinf : &const_ninf);
			cp += 3;
		}
		else
			goto invalid_syntax;

		/*
		 * Check for trailing junk; there should be nothing left but spaces.
		 *
		 * We intentionally do this check before applying the typmod because
		 * we would like to throw any trailing-junk syntax error before any
		 * semantic error resulting from apply_typmod_special().
		 */
		while (*cp)
		{
			if (!isspace((unsigned char) *cp))
				goto invalid_syntax;
			cp++;
		}

		if (!apply_typmod_special(res, typmod, escontext))
			PG_RETURN_NULL();
	}
	else
	{
		/*
		 * We have a normal numeric value, which may be a non-decimal integer
		 * or a regular decimal number.
		 */
		NumericVar	value;
		int			base;
		bool		have_error;

		init_var(&value);

		/*
		 * Determine the number's base by looking for a non-decimal prefix
		 * indicator ("0x", "0o", or "0b").
		 */
		if (cp[0] == '0')
		{
			switch (cp[1])
			{
				case 'x':
				case 'X':
					base = 16;
					break;
				case 'o':
				case 'O':
					base = 8;
					break;
				case 'b':
				case 'B':
					base = 2;
					break;
				default:
					base = 10;
			}
		}
		else
			base = 10;

		/* Parse the rest of the number and apply the sign */
		if (base == 10)
		{
			if (!set_var_from_str(str, cp, &value, &cp, escontext))
				PG_RETURN_NULL();
			value.sign = sign;
		}
		else
		{
			if (!set_var_from_non_decimal_integer_str(str, cp + 2, sign, base,
													  &value, &cp, escontext))
				PG_RETURN_NULL();
		}

		/*
		 * Should be nothing left but spaces. As above, throw any typmod error
		 * after finishing syntax check.
		 */
		while (*cp)
		{
			if (!isspace((unsigned char) *cp))
				goto invalid_syntax;
			cp++;
		}

		if (!apply_typmod(&value, typmod, escontext))
			PG_RETURN_NULL();

		res = make_result_opt_error(&value, &have_error);

		if (have_error)
			ereturn(escontext, (Datum) 0,
					(errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),
					 errmsg("value overflows numeric format")));

		free_var(&value);
	}

	PG_RETURN_NUMERIC(res);

invalid_syntax:
	ereturn(escontext, (Datum) 0,
			(errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
			 errmsg("invalid input syntax for type %s: \"%s\"",
					"numeric", str)));
}
```

代码解析：

```c++
char	   *str = PG_GETARG_CSTRING(0);
```
这里是一个宏，作用是从前端输入的内容里拿到一个char*类型的小数。我们以12323424.567这个数字为例，看看每一步是怎么转换的

```c++
cp = str;
while (*cp)
{
if (!isspace((unsigned char) *cp))
	break;
cp++;
}
```

这个函数就是把12323424.567这个数字赋值给了cp，检查数字中是否有空格，如果没有的话就跳出循环，如果有的话就跳过，一般来说是没有空格的

```c++
numstart = cp;
sign = NUMERIC_POS;

if (*cp == '+')
	cp++;
else if (*cp == '-')
{
	sign = NUMERIC_NEG;
	cp++;
}
```

这里定义了sign，默认情况下，小数是正数，如果检测到了"-"的符号，就把sign改为NUMERIC_NEG，正数的sign是NUMERIC_POS

```c++
if (!isdigit((unsigned char) *cp) && *cp != '.')
{
...
}
else
{
	/*
	 * We have a normal numeric value, which may be a non-decimal integer
	 * or a regular decimal number.
	 */
	NumericVar	value;
	int			base;
	bool		have_error;

	init_var(&value);

	/*
	 * Determine the number's base by looking for a non-decimal prefix
	 * indicator ("0x", "0o", or "0b").
	 */
	if (cp[0] == '0')
	{
		switch (cp[1])
		{
			case 'x':
			case 'X':
				base = 16;
				break;
			case 'o':
			case 'O':
				base = 8;
				break;
			case 'b':
			case 'B':
				base = 2;
				break;
			default:
				base = 10;
		}
	}
	else
		base = 10;

	/* Parse the rest of the number and apply the sign */
	if (base == 10)
	{
		if (!set_var_from_str(str, cp, &value, &cp, escontext))
			PG_RETURN_NULL();
		value.sign = sign;
	}
	else
	{
		if (!set_var_from_non_decimal_integer_str(str, cp + 2, sign, base,
												  &value, &cp, escontext))
			PG_RETURN_NULL();
	}

	/*
	 * Should be nothing left but spaces. As above, throw any typmod error
	 * after finishing syntax check.
	 */
	while (*cp)
	{
		if (!isspace((unsigned char) *cp))
			goto invalid_syntax;
		cp++;
	}

	if (!apply_typmod(&value, typmod, escontext))
		PG_RETURN_NULL();

	res = make_result_opt_error(&value, &have_error);

	if (have_error)
		ereturn(escontext, (Datum) 0,
				(errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),
				 errmsg("value overflows numeric format")));

	free_var(&value);
}
```

这一段超级长的判断代码，直接看else部分即可，if部分就是检查NaN和infinities的，else部分就是输入的正常数字的解析。
首先定义了一个NumericVar	value;NumericVar就是在内存中，对整个数字的说明：
ndigits：指的digits数组元素的个数
weight：表示的是整数部分所占用的数组元素个数
sign：这是一个通过计算得出来的值，后面有计算方式
dscale：小数部分的位数
digits：这是具体的数值
```c++
typedef struct NumericVar
{
	int			ndigits;		/* # of digits in digits[] - can be 0! */
	int			weight;			/* weight of first digit */
	int			sign;			/* NUMERIC_POS, _NEG, _NAN, _PINF, or _NINF */
	int			dscale;			/* display scale */
	NumericDigit *buf;			/* start of palloc'd space for digits[] */
	NumericDigit *digits;		/* base-NBASE digits */
} NumericVar;
```

首先先初始化value
```c++
#define init_var(v)		memset(v, 0, sizeof(NumericVar))
```

检查进制，也就是检查cp的第一位，默认为10
```c++
if (cp[0] == '0')
{
	switch (cp[1])
	{
		case 'x':
		case 'X':
			base = 16;
			break;
		case 'o':
		case 'O':
			base = 8;
			break;
		case 'b':
		case 'B':
			base = 2;
			break;
		default:
			base = 10;
	}
}
else
	base = 10;
```

如果为10进制，就调用set_var_from_str函数，这个函数是解析numeric的核心函数，主要构建了NumericVar的内容。这个函数后面解析，先把主函数看完

```c++
if (base == 10)
{
	if (!set_var_from_str(str, cp, &value, &cp, escontext))
		PG_RETURN_NULL();
	value.sign = sign;
}
```

这后面的函数也很简单，就是再次检查空格，make_result_opt_error函数的作用是把内存中的NumericVar写入到磁盘上

```c++
while (*cp)
{
	if (!isspace((unsigned char) *cp))
		goto invalid_syntax;
	cp++;
}

if (!apply_typmod(&value, typmod, escontext))
	PG_RETURN_NULL();

res = make_result_opt_error(&value, &have_error);

if (have_error)
	ereturn(escontext, (Datum) 0,
			(errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),
			 errmsg("value overflows numeric format")));

free_var(&value);
```

现在来看看set_var_from_str这个函数，下面是总体的内容，我们来一步一步解析

```c++
static bool
set_var_from_str(const char *str, const char *cp,
				 NumericVar *dest, const char **endptr,
				 Node *escontext)
{
	bool		have_dp = false;
	int			i;
	unsigned char *decdigits;
	int			sign = NUMERIC_POS;
	int			dweight = -1;
	int			ddigits;
	int			dscale = 0;
	int			weight;
	int			ndigits;
	int			offset;
	NumericDigit *digits;

	/*
	 * We first parse the string to extract decimal digits and determine the
	 * correct decimal weight.  Then convert to NBASE representation.
	 */
	switch (*cp)
	{
		case '+':
			sign = NUMERIC_POS;
			cp++;
			break;

		case '-':
			sign = NUMERIC_NEG;
			cp++;
			break;
	}

	if (*cp == '.')
	{
		have_dp = true;
		cp++;
	}

	if (!isdigit((unsigned char) *cp))
		goto invalid_syntax;

	decdigits = (unsigned char *) palloc(strlen(cp) + DEC_DIGITS * 2);

	/* leading padding for digit alignment later */
	memset(decdigits, 0, DEC_DIGITS);
	i = DEC_DIGITS;

	while (*cp)
	{
		if (isdigit((unsigned char) *cp))
		{
			decdigits[i++] = *cp++ - '0';
			if (!have_dp)
				dweight++;
			else
				dscale++;
		}
		else if (*cp == '.')
		{
			if (have_dp)
				goto invalid_syntax;
			have_dp = true;
			cp++;
			/* decimal point must not be followed by underscore */
			if (*cp == '_')
				goto invalid_syntax;
		}
		else if (*cp == '_')
		{
			/* underscore must be followed by more digits */
			cp++;
			if (!isdigit((unsigned char) *cp))
				goto invalid_syntax;
		}
		else
			break;
	}

	ddigits = i - DEC_DIGITS;
	/* trailing padding for digit alignment later */
	memset(decdigits + i, 0, DEC_DIGITS - 1);

	/* Handle exponent, if any */
	if (*cp == 'e' || *cp == 'E')
	{
		int64		exponent = 0;
		bool		neg = false;

		/*
		 * At this point, dweight and dscale can't be more than about
		 * INT_MAX/2 due to the MaxAllocSize limit on string length, so
		 * constraining the exponent similarly should be enough to prevent
		 * integer overflow in this function.  If the value is too large to
		 * fit in storage format, make_result() will complain about it later;
		 * for consistency use the same ereport errcode/text as make_result().
		 */

		/* exponent sign */
		cp++;
		if (*cp == '+')
			cp++;
		else if (*cp == '-')
		{
			neg = true;
			cp++;
		}

		/* exponent digits */
		if (!isdigit((unsigned char) *cp))
			goto invalid_syntax;

		while (*cp)
		{
			if (isdigit((unsigned char) *cp))
			{
				exponent = exponent * 10 + (*cp++ - '0');
				if (exponent > PG_INT32_MAX / 2)
					goto out_of_range;
			}
			else if (*cp == '_')
			{
				/* underscore must be followed by more digits */
				cp++;
				if (!isdigit((unsigned char) *cp))
					goto invalid_syntax;
			}
			else
				break;
		}

		if (neg)
			exponent = -exponent;

		dweight += (int) exponent;
		dscale -= (int) exponent;
		if (dscale < 0)
			dscale = 0;
	}

	/*
	 * Okay, convert pure-decimal representation to base NBASE.  First we need
	 * to determine the converted weight and ndigits.  offset is the number of
	 * decimal zeroes to insert before the first given digit to have a
	 * correctly aligned first NBASE digit.
	 */
	if (dweight >= 0)
		weight = (dweight + 1 + DEC_DIGITS - 1) / DEC_DIGITS - 1;
	else
		weight = -((-dweight - 1) / DEC_DIGITS + 1);
	offset = (weight + 1) * DEC_DIGITS - (dweight + 1);
	ndigits = (ddigits + offset + DEC_DIGITS - 1) / DEC_DIGITS;

	alloc_var(dest, ndigits);
	dest->sign = sign;
	dest->weight = weight;
	dest->dscale = dscale;

	i = DEC_DIGITS - offset;
	digits = dest->digits;

	while (ndigits-- > 0)
	{
#if DEC_DIGITS == 4
		*digits++ = ((decdigits[i] * 10 + decdigits[i + 1]) * 10 +
					 decdigits[i + 2]) * 10 + decdigits[i + 3];
#elif DEC_DIGITS == 2
		*digits++ = decdigits[i] * 10 + decdigits[i + 1];
#elif DEC_DIGITS == 1
		*digits++ = decdigits[i];
#else
#error unsupported NBASE
#endif
		i += DEC_DIGITS;
	}

	pfree(decdigits);

	/* Strip any leading/trailing zeroes, and normalize weight if zero */
	strip_var(dest);

	/* Return end+1 position for caller */
	*endptr = cp;

	return true;

out_of_range:
	ereturn(escontext, false,
			(errcode(ERRCODE_NUMERIC_VALUE_OUT_OF_RANGE),
			 errmsg("value overflows numeric format")));

invalid_syntax:
	ereturn(escontext, false,
			(errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
			 errmsg("invalid input syntax for type %s: \"%s\"",
					"numeric", str)));
}
```

首先是初始化了一堆变量，之后循环cp这个数值的指针，这里还是对sign进行定义，如果为正数就设置为NUMERIC_POS，如果负数就设置为NUMERIC_NEG。然后让cp++，指针指向cp[1]

```c++
switch (*cp)
{
	case '+':
		sign = NUMERIC_POS;
		cp++;
		break;

	case '-':
		sign = NUMERIC_NEG;
		cp++;
		break;
}
```

如果cp的指针内容是"."的话，就把have_dp设置为true，然后cp的指针指向下一个值，这里也比较清楚have_dp的作用了，就是记录小数点的

```c++
if (*cp == '.')
{
	have_dp = true;
	cp++;
}
```

这里是比较核心的部分了，主要是循环cp数值的内容，这段代码的作用就是记录dweight和dscale的值的。
dweight是整数部分的位数，因为初始值为-1，所以得出的位数是比真实的位数少1位的。
dscale是小数部分的位数

```c++
while (*cp)
{
	if (isdigit((unsigned char) *cp))
	{
		decdigits[i++] = *cp++ - '0';
		if (!have_dp)
			dweight++;
		else
			dscale++;
	}
	else if (*cp == '.')
	{
		if (have_dp)
			goto invalid_syntax;
		have_dp = true;
		cp++;
		/* decimal point must not be followed by underscore */
		if (*cp == '_')
			goto invalid_syntax;
	}
	else if (*cp == '_')
	{
		/* underscore must be followed by more digits */
		cp++;
		if (!isdigit((unsigned char) *cp))
			goto invalid_syntax;
	}
	else
		break;
}
```

ddigits是记录数值的有效位数的，这里是先重置为0，前面有把DEC_DIGITS赋值给i，DEC_DIGITS宏定义为4

```c++
i = DEC_DIGITS;
ddigits = i - DEC_DIGITS;
```

科学计数法这段就先不看了

好了，这部分就是把内存中NumericVar结构体的内容填充进去，NumericVar结构体说明在前面有

```c++
if (dweight >= 0)
	weight = (dweight + 1 + DEC_DIGITS - 1) / DEC_DIGITS - 1;
else
	weight = -((-dweight - 1) / DEC_DIGITS + 1);
offset = (weight + 1) * DEC_DIGITS - (dweight + 1);
ndigits = (ddigits + offset + DEC_DIGITS - 1) / DEC_DIGITS;

alloc_var(dest, ndigits);
dest->sign = sign;
dest->weight = weight;
dest->dscale = dscale;

i = DEC_DIGITS - offset;
digits = dest->digits;
```










































































