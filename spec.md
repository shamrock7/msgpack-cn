# MessagePack 规范

> [5839f04a64738ae9a32167a17b1be4ab11ceb452](https://github.com/msgpack/msgpack/commit/5839f04a64738ae9a32167a17b1be4ab11ceb452)

MessagePack 是一种类似于 JSON 的 对象序列化规范。

MessagePack 的两个概念：**类型系统(type system)**和**格式(formats)**。

序列化(Serialization) 是指将应用对象通过 MessagePack 类型系统转成 MessagePack 格式。

反序列化(Deserialization)是指将 MessagePack 格式通过 MessagePack 类型系统转回 应用对象。


```
    序列化:
        应用对象
        -->  MessagePack 类型系统
        -->  MessagePack 格式(字节数组) 

    反序列化:
        MessagePack 格式(字节数组)
        -->  MessagePack 类型系统
        -->  应用对象

```

该文档描述了 MessagePack 类型系统、MessagePack 格式以及如何转换他们。

## 目录

* MessagePack 规范
  * [类型系统](#type-system)
      * [限制](#limitation)
      * [扩展类型](#extension-types)
  * [格式](#formats)
      * [概要](#overview)
      * [符号表](#notation-in-diagrams)
      * [nil 格式](#nil-format)
      * [bool 格式簇](#bool-format-family)
      * [int 格式簇](#int-format-family)
      * [float 格式簇](#float-format-family)
      * [str 格式簇](#str-format-family)
      * [bin 格式簇](#bin-format-family)
      * [array 格式簇](#array-format-family)
      * [map 格式簇](#map-format-family)
      * [ext 格式簇](#ext-format-family)
      * [时间戳扩展类型](#timestamp-extension-type)
  * [序列化: 数据类型到 MessagePack 格式的转换](#serialization-type-to-format-conversion)
  * [反序列化: MessagePack 格式到数据类型的转换](#deserialization-format-to-type-conversion)
  * [未来的讨论](#future-discussion)
    * [概述](#profile)
  * [实现指南](#implementation-guidelines)
    * [更新 MessagePack 规范](#upgrading-messagepack-specification)

## 类型系统(Type system)

* 类型
  * **Integer** 代表整数(integer)
  * **Nil** 代表空类型(nil)
  * **Boolean** 代表 true 或 false
  * **Float** 代表一个 IEEE 754 双精度浮点数，包括 非数字(NaN) 和 无穷大(Infinity)
  * **Raw**
      * **String** 扩展原生(Raw)类型，表示一个 UTF-8 的字符串
      * **Binary** 扩展原生(Raw)类型，表示一个字节数组
  * **Array** 代表对象数组
  * **Map** 代表对象的键值对(key-value pairs)
  * **Extension** 表示类型信息和字节数组的组合，其中类型信息由整数指定，意为其由应用或 MessagePack 定义。
      * **Timestamp** 代表一个独立于时区和日历的世界时时间戳，最大精度为纳秒。

### 限制(Limitation)

* 整数的限制是 `-(2^63)` 到 `(2^64)-1`
* 二进制对象的最大字节长度是 `(2^32)-1`
* 字符串对象的最大字节长度是 `(2^32)-1`
* 字符串对象可能包含无效的字节序列，当前接收到无效的字节序列时，反序列化的行为依赖于实际的实现
    * 反序列化应该提供获取原生字节数组的功能，以便应用能够决定如何处理该对象
* 数组的最大长度是 `(2^32)-1`
* Map 键值对的最大键值对数是 `(2^32)-1`

### 扩展类型(Extension types)

MessagePack 允许应用通过扩展类型定义应用特定的类型。
扩展类型包括一个表示类型类别的整数以及一个存储数据字节数组。

应用可以使用 `0` 到 `127` 来存储应用特定的类型信息。比如应用定义 `type = 0` 表示应用特定的类型系统，并且在载荷(payload)中存储类型的名称以及类型的值。


MessagePack 保留了 `-1` 到 `-128`，为将来添加预定义的类型进行扩展。这些类型会被用于替换更多类型，而无需在不同的编程环境中使用预共享的静态类型模式。

```

    [0, 127]: 应用特定类型
    [-128, -1]: 为预定义的类型保留
```

由于扩展类型可能会增加，所以老应用可能并没完全实现这些类型。但是，老应用仍然可以将这样的类型作为一种扩展类型进行处理。因此，应用可以决定是拒绝未知扩展类型还是接收未知类型，或者在不触及载荷的情况下传给另一个应用。

下面是预定义类型列表。类型格式在 [格式(Formats)](#formats-timestamp)部分已经定义好。

| 名称 | 类型 |
| --------- | ---- |
| Timestamp | -1 |

## 格式(Formats)

### 概述

格式名称     | 首字节(二进制) | 首字节(十六进制)
--------------- | ---------------------- | -------------------
positive fixint | 0xxxxxxx               | 0x00 - 0x7f
fixmap          | 1000xxxx               | 0x80 - 0x8f
fixarray        | 1001xxxx               | 0x90 - 0x9f
fixstr          | 101xxxxx               | 0xa0 - 0xbf
nil             | 11000000               | 0xc0
(never used)    | 11000001               | 0xc1
false           | 11000010               | 0xc2
true            | 11000011               | 0xc3
bin 8           | 11000100               | 0xc4
bin 16          | 11000101               | 0xc5
bin 32          | 11000110               | 0xc6
ext 8           | 11000111               | 0xc7
ext 16          | 11001000               | 0xc8
ext 32          | 11001001               | 0xc9
float 32        | 11001010               | 0xca
float 64        | 11001011               | 0xcb
uint 8          | 11001100               | 0xcc
uint 16         | 11001101               | 0xcd
uint 32         | 11001110               | 0xce
uint 64         | 11001111               | 0xcf
int 8           | 11010000               | 0xd0
int 16          | 11010001               | 0xd1
int 32          | 11010010               | 0xd2
int 64          | 11010011               | 0xd3
fixext 1        | 11010100               | 0xd4
fixext 2        | 11010101               | 0xd5
fixext 4        | 11010110               | 0xd6
fixext 8        | 11010111               | 0xd7
fixext 16       | 11011000               | 0xd8
str 8           | 11011001               | 0xd9
str 16          | 11011010               | 0xda
str 32          | 11011011               | 0xdb
array 16        | 11011100               | 0xdc
array 32        | 11011101               | 0xdd
map 16          | 11011110               | 0xde
map 32          | 11011111               | 0xdf
negative fixint | 111xxxxx               | 0xe0 - 0xff

### 符号表

    单字节:
    +--------+
    |        |
    +--------+

    多字节:
    +========+
    |        |
    +========+

    在 MessagePack 中存储的多个对象:
    +~~~~~~~~~~~~~~~~~+
    |                 |
    +~~~~~~~~~~~~~~~~~+

`X`、 `Y`、 `Z` 和 `A` 将是被实际 bit 替换的符号。

### nil 格式

Nil 格式在一个字节中存储 nil

    nil:
    +--------+
    |  0xc0  |
    +--------+

### bool 格式簇

Bool 格式簇在一个字节中存储

    false:
    +--------+
    |  0xc2  |
    +--------+

    true:
    +--------+
    |  0xc3  |
    +--------+

### int 格式簇

Int 格式簇在 1、2、3、5 或 9 个字节中存储一个整数

    positive fixint 存储 7-bit 无符号整数
    +--------+
    |0XXXXXXX|
    +--------+

    negative fixint 存储 5-bit 有符号整数
    +--------+
    |111YYYYY|
    +--------+

    * 0XXXXXXX 是 8-bit 无符号整数
    * 111YYYYY 是 8-bit 有符号整数

    uint 8 存储 8-bit 无符号整数
    +--------+--------+
    |  0xcc  |ZZZZZZZZ|
    +--------+--------+

    uint 16 存储 16-bit 大端(big-endian)无符号整数
    +--------+--------+--------+
    |  0xcd  |ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+

    uint 32 存储 32-bit 大端无符号整数
    +--------+--------+--------+--------+--------+
    |  0xce  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+--------+--------+

    uint 64 s存储 64-bit 大端无符号整数
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xcf  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    int 8 存储 8-bit 有符号整数
    +--------+--------+
    |  0xd0  |ZZZZZZZZ|
    +--------+--------+

    int 16 存储 16-bit 大端有符号整数
    +--------+--------+--------+
    |  0xd1  |ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+

    int 32 存储 32-bit 大端有符号整数
    +--------+--------+--------+--------+--------+
    |  0xd2  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+--------+--------+

    int 64 存储 64-bit 大端有符号整数
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xd3  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

### float 格式簇

Float 格式簇在 5字节或9字节中存储浮点数。

    float 32 存储 IEEE 754 单精度浮点数格式的浮点数:
    +--------+--------+--------+--------+--------+
    |  0xca  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+

    float 64 存储 IEEE 754 双精度浮点数格式的浮点数:
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xcb  |YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|YYYYYYYY|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+

    其中
    * XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX 是大端 IEEE 754 单精度浮点数。
      从单精度到双精度的扩展并不损失精度。
    * YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY_YYYYYYYY 是 大端 IEEE 754 双精度浮点数

### str 格式簇

str 格式簇存储一个字节数组，在额外1、2、3或5字节中存储字节数组的大小。

    fixstr 存储一个长度最多 31 的字节数组:
    +--------+========+
    |101XXXXX|  data  |
    +--------+========+

    str 8 存储一个长度最多 2^8 -1 的字节数组:
    +--------+--------+========+
    |  0xd9  |YYYYYYYY|  data  |
    +--------+--------+========+

    str 16 存储一个长度最多 2^16 - 1 的字节数组:
    +--------+--------+--------+========+
    |  0xda  |ZZZZZZZZ|ZZZZZZZZ|  data  |
    +--------+--------+--------+========+

    str 32 存储一个长度最多 2^32 - 1 的字节数组:
    +--------+--------+--------+--------+--------+========+
    |  0xdb  |AAAAAAAA|AAAAAAAA|AAAAAAAA|AAAAAAAA|  data  |
    +--------+--------+--------+--------+--------+========+

    其中
    * XXXXX 是一个代表 N 的 5-bit 的无符号整数
    * YYYYYYYY 是一个代表 N 的 8-bit 的无符号整数
    * ZZZZZZZZ\_ZZZZZZZZ 是一个代表 N 的大端存储的 16-bit 的无符号整数
    * AAAAAAAA\_AAAAAAAA\_AAAAAAAA\_AAAAAAAA 是一个代表 N 的大端存储的 32-bit 的无符号整数
    * 其中 N 表示数据字节长度

### 二进制数据格式簇

二进制数据格式簇存储一个字节数组，并在额外的 2、3 或 5 字节中存储字节数组长度

    bin 8 存储一个长度最多 2^8 - 1 的字节数组:
    +--------+--------+========+
    |  0xc4  |XXXXXXXX|  data  |
    +--------+--------+========+

    bin 16 存储一个长度最多 2^16 - 1 的字节数组:
    +--------+--------+--------+========+
    |  0xc5  |YYYYYYYY|YYYYYYYY|  data  |
    +--------+--------+--------+========+

    bin 32 存储一个长度最多 2^32 - 1 的字节数组:
    +--------+--------+--------+--------+--------+========+
    |  0xc6  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|  data  |
    +--------+--------+--------+--------+--------+========+

    其中
    * XXXXXXXX 是代表 N 的 8-bit 无符号整数
    * YYYYYYYY_YYYYYYYY 是代表 N 的大端存储的 16-bit 无符号整数
    * ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ_ZZZZZZZZ 是代表 N 的大端存储的 32-bit 无符号整数
    * 其中 N 表示数据长度

### array 格式簇

array 格式簇存储一个元素序列，并在额外的 1、3 和 5 字节中存储元素个数。

    fixarray 存储一个最多 15 个元素的数组:
    +--------+~~~~~~~~~~~~~~~~~+
    |1001XXXX|    N objects    |
    +--------+~~~~~~~~~~~~~~~~~+

    array 16 存储一个最多 2^16 - 1 个元素的数组:
    +--------+--------+--------+~~~~~~~~~~~~~~~~~+
    |  0xdc  |YYYYYYYY|YYYYYYYY|    N objects    |
    +--------+--------+--------+~~~~~~~~~~~~~~~~~+

    array 32 存储一个最多 2^32 - 1 个元素的数组:
    +--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+
    |  0xdd  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|    N objects    |
    +--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+

    其中
    * XXXX 是一个表示 N 的 4-bit 无符号整数
    * YYYYYYYY\_YYYYYYYY 是一个表示 N 的大端存储的 16-bit 无符号整数
    * ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ 是一个表示 N 的大端存储的 32-bit 无符号整数
    * 其中 N 表示数组大小

### map 格式簇

map 格式簇存储一系列的 key-value 对，并在额外的1、3、5字节中存储 key-value 的个数。

    fixmap 存储一个最多 15 个元素的 map
    +--------+~~~~~~~~~~~~~~~~~+
    |1000XXXX|   N*2 objects   |
    +--------+~~~~~~~~~~~~~~~~~+

    map 16 存储一个最多 2^16-1 个元素的 map

    +--------+--------+--------+~~~~~~~~~~~~~~~~~+
    |  0xde  |YYYYYYYY|YYYYYYYY|   N*2 objects   |
    +--------+--------+--------+~~~~~~~~~~~~~~~~~+

    map 32 存储一个最多 2^32 - 1 个元素的 map
    +--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+
    |  0xdf  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|   N*2 objects   |
    +--------+--------+--------+--------+--------+~~~~~~~~~~~~~~~~~+

    其中
    * XXXX 是一个表示 N 的 4-bit 无符号整数
    * YYYYYYYY\_YYYYYYYY 是一个表示 N 的大端存储的 16-bit 无符号整数
    * ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ 是一个表示 N 的大端存储的 32-bit 无符号整数
    * 其中 N 表示 map 的大小
    * 对象中的奇数元素(下标为奇数的元素)表示 map 的 key 
    * 接着的下一个元素(下标为偶数的元素)表示 key 对应的值

### ext 格式簇

ext 格式簇存储整数和字节数组的元组。

    fixext 1 存储一个整数和一个长度是 1 的字节数组
    +--------+--------+--------+
    |  0xd4  |  type  |  data  |
    +--------+--------+--------+

    fixext 2 存储一个整数和一个长度是 2 的字节数组
    +--------+--------+--------+--------+
    |  0xd5  |  type  |       data      |
    +--------+--------+--------+--------+

    fixext 4 存储一个整数和一个长度是 4 的字节数组
    +--------+--------+--------+--------+--------+--------+
    |  0xd6  |  type  |                data               |
    +--------+--------+--------+--------+--------+--------+

    fixext 8 存储一个整数和一个长度是 8 的字节数组
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xd7  |  type  |                                  data                                 |
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+

    fixext 16 存储一个整数和一个长度是 16 的字节数组
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xd8  |  type  |                                  data                                  
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
    +--------+--------+--------+--------+--------+--------+--------+--------+
                                  data (cont.)                              |
    +--------+--------+--------+--------+--------+--------+--------+--------+

    ext 8 存储一个整数和一个长度最多 2^8 - 1 的字节数组
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+--------+
    +--------+--------+--------+========+
    |  0xc7  |XXXXXXXX|  type  |  data  |
    +--------+--------+--------+========+

    ext 16 存储一个整数和一个长度最多 2^16 - 1 的字节数组
    +--------+--------+--------+--------+========+
    |  0xc8  |YYYYYYYY|YYYYYYYY|  type  |  data  |
    +--------+--------+--------+--------+========+

    ext 32 存储一个整数和一个长度最多 2^32 - 1 的字节数组
    +--------+--------+--------+--------+--------+--------+========+
    |  0xc9  |ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|ZZZZZZZZ|  type  |  data  |
    +--------+--------+--------+--------+--------+--------+========+

    其中
    * XXXXXXXX 是一个表示 N 的 8-bit 的无符号整数
    * YYYYYYYY\_YYYYYYYY 是一个表示 N 的 大端存储的 16-bit 的无符号整数
    * ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ\_ZZZZZZZZ 是一个表示 N 的大端存储 32-bit 无符号整数
    * 其中 N 表示数据长度
    * type 是一个 8-bit 的有符号整数
    * type < 0 为以后的扩展类型保留，包括 2 字节类型信息

### 时间戳扩展类型

时间戳扩展类型已经分配给扩展类型 `-1`。其定义了三种格式: 32 位格式、64 位格式以及 96 位格式。

    
    timestamp 32 用 32-bit 无符号整数存储自 1970-01-01 00:00:00 UTC 以来过去的秒数:
    +--------+--------+--------+--------+--------+--------+
    |  0xd6  |   -1   |   seconds in 32-bit unsigned int  |
    +--------+--------+--------+--------+--------+--------+

    timestamp 64 用 32-bit 无符号整数存储自 1970-01-01 00:00:00 UTC 以来过去的纳秒数:
    +--------+--------+--------+--------+--------+------|-+--------+--------+--------+--------+
    |  0xd7  |   -1   | nanosec. in 30-bit unsigned int |   seconds in 34-bit unsigned int    |
    +--------+--------+--------+--------+--------+------^-+--------+--------+--------+--------+

    timestamp 96 用 64-bit 有符号整数和 32-bit 无符号整数分别存储自 1970-01-01 00:00:00 UTC 以来过去的纳秒数和秒数:
    +--------+--------+--------+--------+--------+--------+--------+
    |  0xc7  |   12   |   -1   |nanoseconds in 32-bit unsigned int |
    +--------+--------+--------+--------+--------+--------+--------+
    +--------+--------+--------+--------+--------+--------+--------+--------+
                        seconds in 64-bit signed int                        |
    +--------+--------+--------+--------+--------+--------+--------+--------+

* Timestamp 32 format 可以表示一个 [1970-01-01 00:00:00 UTC, 2106-02-07 06:28:16 UTC) 范围内的时间戳。纳秒部分为0。
* Timestamp 64 format 可以表示一个 [1970-01-01 00:00:00.000000000 UTC, 2514-05-30 01:53:04.000000000 UTC) 范围内的时间戳。
* Timestamp 96 format 可以表示一个 [-292277022657-01-27 08:29:52 UTC, 292277026596-12-04 15:30:08.000000000 UTC) 范围内的时间戳。
* 在 timestamp 64 and timestamp 96 格式中，纳秒必须小于等于 999999999。

序列化的伪代码:

    struct timespec {
        long tv_sec;  // 秒
        long tv_nsec; // 纳秒
    } time;
    if ((time.tv_sec >> 34) == 0) {
        uint64_t data64 = (time.tv_nsec << 34) | time.tv_sec;
        if (data64 & 0xffffffff00000000L == 0) {
            // timestamp 32
            uint32_t data32 = data64;
            serialize(0xd6, -1, data32)
        }
        else {
            // timestamp 64
            serialize(0xd7, -1, data64)
        }
    }
    else {
        // timestamp 96
        serialize(0xc7, 12, -1, time.tv_nsec, time.tv_sec)
    }

反序列化的伪代码:

     ExtensionValue value = deserialize_ext_type();
     struct timespec result;
     switch(value.length) {
     case 4:
         uint32_t data32 = value.payload;
         result.tv_nsec = 0;
         result.tv_sec = data32;
     case 8:
         uint64_t data64 = value.payload;
         result.tv_nsec = data64 >> 34;
         result.tv_sec = data64 & 0x00000003ffffffffL;
     case 12:
         uint32_t data32 = value.payload;
         uint64_t data64 = value.payload + 4;
         result.tv_nsec = data32;
         result.tv_sec = data64;
     default:
         // error
     }

## 序列化：类型到格式的转换

MessagePack 序列化器将 MessagePack 类型转换为如下格式:

源类型       | 输出格式
------------ | ---------------------------------------------------------------------------------------
Integer      | int 格式簇 (positive fixint, negative fixint, int 8/16/32/64 or uint 8/16/32/64)
Nil          | nil
Boolean      | bool 格式簇 (false or true)
Float        | float 格式簇 (float 32/64)
String       | str 格式簇 (fixstr or str 8/16/32)
Binary       | bin 格式簇 (bin 8/16/32)
Array        | array 格式簇 (fixarray or array 16/32)
Map          | map 格式簇 (fixmap or map 16/32)
Extension    | ext 格式簇 (fixext or ext 8/16/32)

如果一个对象可以被表示为多种可能的输出格式，序列化器`应该`使用更少字节数的格式表示该数据。

## 反序列化：格式到类型的转换

MessagePack 反序列化器将 MessagePack 格式转换为如下类型：

源格式                                                               | 输出类型
-------------------------------------------------------------------- | -----------
positive fixint, negative fixint, int 8/16/32/64 and uint 8/16/32/64 | Integer
nil                                                                  | Nil
false and true                                                       | Boolean
float 32/64                                                          | Float
fixstr and str 8/16/32                                               | String
bin 8/16/32                                                          | Binary
fixarray and array 16/32                                             | Array
fixmap map 16/32                                                     | Map
fixext and ext 8/16/32                                               | Extension

## 未来讨论

### Profile

Profile 是应用程序在共享相同语法以使 MessagePack 适应某些用例的同时，限制 MessagePack 语义的一种想法。

例如：应用可以移除二进制类型，限制 map 对象的 key 只能是字符串类型，以及保留一些约束以让语法与 JSON 兼容。使用该规范的应用可能移除字符串类型和二进制类型，并且将字节数组视为原始数据。应用使用序列化数据哈希（数字）可以对 map 的 key进行排序，以生成确定性的序列化数据。

## 实现指南

### MessagePack 规范更新

MessagePack 规范在此时变动。下面是更新已存在的 MessagePack 实现的概览：

* 在小版本中，反序列化器支持 bin 格式簇和 str 8 格式。反序列化对象的类型应该与 raw 16（等于 str 16）或 raw 32（等于 str 32）
* 在主版本中，序列化器用 bin 格式簇 和 str 格式簇区分二进制类型和字符串类型。
  * 同时，序列号器应支持不使用 bin 格式簇 和 str 8 格式的‘兼容模式’。


___

    MessagePack 规范
    最后修改于 2017-08-09 22:42:07 -0700
    Sadayuki Furuhashi © 2013-04-21 21:52:33 -0700
