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
