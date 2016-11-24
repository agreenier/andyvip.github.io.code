---
date: 2016-11-23 23:03
title: atomic类
tags: [android,atomic]
categories: [android,api]
---

#atomic是什么
#atomic的一个例子
```C++
atomic_uint_least32_t serial
atomic_init(&serial, 0);
```
#atomic代码解读
-   atomic变量的数据结构
-   atomic的相关操作

```
#define _Atomic(T)              struct { T volatile __val; }
typedef _Atomic(bool)			atomic_bool;
typedef _Atomic(char)			atomic_char;
typedef _Atomic(signed char)		atomic_schar;
typedef _Atomic(unsigned char)		atomic_uchar;
typedef _Atomic(short)			atomic_short;
typedef _Atomic(unsigned short)		atomic_ushort;
typedef _Atomic(int)			atomic_int;
typedef _Atomic(unsigned int)		atomic_uint;
typedef _Atomic(long)			atomic_long;
typedef _Atomic(unsigned long)		atomic_ulong;
typedef _Atomic(long long)		atomic_llong;
typedef _Atomic(unsigned long long)	atomic_ullong;
#if __STDC_VERSION__ >= 201112L || __cplusplus >= 201103L
  typedef _Atomic(char16_t)		atomic_char16_t;
  typedef _Atomic(char32_t)		atomic_char32_t;
#endif
typedef _Atomic(wchar_t)		atomic_wchar_t;
typedef _Atomic(int_least8_t)		atomic_int_least8_t;
typedef _Atomic(uint_least8_t)	atomic_uint_least8_t;
typedef _Atomic(int_least16_t)	atomic_int_least16_t;
typedef _Atomic(uint_least16_t)	atomic_uint_least16_t;
typedef _Atomic(int_least32_t)	atomic_int_least32_t;
typedef _Atomic(uint_least32_t)	atomic_uint_least32_t;
typedef _Atomic(int_least64_t)	atomic_int_least64_t;
typedef _Atomic(uint_least64_t)	atomic_uint_least64_t;
typedef _Atomic(int_fast8_t)		atomic_int_fast8_t;
typedef _Atomic(uint_fast8_t)		atomic_uint_fast8_t;
typedef _Atomic(int_fast16_t)		atomic_int_fast16_t;
typedef _Atomic(uint_fast16_t)	atomic_uint_fast16_t;
typedef _Atomic(int_fast32_t)		atomic_int_fast32_t;
typedef _Atomic(uint_fast32_t)	atomic_uint_fast32_t;
typedef _Atomic(int_fast64_t)		atomic_int_fast64_t;
typedef _Atomic(uint_fast64_t)	atomic_uint_fast64_t;
typedef _Atomic(intptr_t)		atomic_intptr_t;
typedef _Atomic(uintptr_t)		atomic_uintptr_t;
typedef _Atomic(size_t)		atomic_size_t;
typedef _Atomic(ptrdiff_t)		atomic_ptrdiff_t;
typedef _Atomic(intmax_t)		atomic_intmax_t;
typedef _Atomic(uintmax_t)		atomic_uintmax_t;
