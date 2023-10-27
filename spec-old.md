# MessagePack 格式规范

> [acbcdf6b2a5a62666987c041124a10c69124be0d](https://github.com/msgpack/msgpack/commit/acbcdf6b2a5a62666987c041124a10c69124be0d)

**该规范已经更新过。查看更新过的版本 [spec.md](spec.md) **


MessagePack 在序列化的数据中保存了类型信息。因此，每一种数据都会存储成 **type-data** 或 **type-length-data** 的风格。
MessagePack 支持如下类型：

* 定长类型
  * 整数
  * nil
  * 布尔
  * 浮点数
* 变长类型
  * 原始字节
* 容器类型
  * 数组
  * Map

每一种类型都有一种或多种序列化的格式：

* 定长类型
  * 整数
      * positive fixnum
      * negative fixnum
      * uint 8
      * uint 16
      * uint 32
      * uint 64
      * int 8
      * int 16
      * int 32
      * int 64
  * Nil
      * nil
  * 布尔
      * true
      * false
  * 浮点数
      * float
      * double
* 变长类型
  * 原始字节
      * fix raw
      * raw 16
      * raw 32
* 容器类型
  * 数组
      * fix array
      * array 16
      * array 32
  * Map
      * fix map
      * map 16
      * map 32

字符串是用 UTF-8 编码和 Raw 类型序列化的。

看下 [msgpack/msgpack#121]](https://github.com/msgpack/msgpack/issues/121) 这个 issue 以理解为什么 msgpack 并没有字符串类型

## 整数

### positive fixnum

在一个字节中存储一个 [0, 127] 范围内的整数

    +--------+
    |0XXXXXXX|
    +--------+
    => unsigned 8-bit 0XXXXXXX


### negative fixnum

在一个字节中存储一个 [-32, -1] 范围内的整数

    +--------+
    |111XXXXX|
    +--------+
    => signed 8-bit 111XXXXX


### uint 8

在 2 个字节中存储一个无符号的 8-bit 整数

    +--------+--------+
    |  0xcc  |XXXXXXXX|
    +--------+--------+
    => unsigned 8-bit XXXXXXXX


### uint 16

在 3 个字节中存储一个无符号的 16-bit 大端存储的整数

    +--------+--------+--------+
    |  0xcd  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+
    => unsigned 16-bit big-endian XXXXXXXX_XXXXXXXX


### uint 32

在 5 个字节中存储一个无符号的 32-bit 大端存储的整数

    +--------+--------+--------+--------+--------+
    |  0xce  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+
    => unsigned 32-bit big-endian XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX


### uint 64

在 9 个字节中存储一个无符号的 64-bit 大端存储的整数

    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xcf  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    => unsigned 64-bit big-endian XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX


### int 8

在 2 个字节中存储一个有符号的 8-bit 整数

    +--------+--------+
    |  0xd0  |XXXXXXXX|
    +--------+--------+
    => signed 8-bit XXXXXXXX

### int 16

在 3 个字节中存储一个有符号的 16-bit 大端存储的整数

    +--------+--------+--------+
    |  0xd1  |XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+
    => signed 16-bit big-endian XXXXXXXX_XXXXXXXX


### int 32

在 5 个字节中存储一个有符号的 32-bit 大端存储的整数

    +--------+--------+--------+--------+--------+
    |  0xd2  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+
    => signed 32-bit big-endian XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX

### int 64

在 9 个字节中存储一个有符号的 64-bit 大端存储的整数

    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xd3  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    => signed 64-bit big-endian XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX


## Nil


### nil

存储一个 nil 。

    +--------+
    |  0xc0  |
    +--------+


## Boolean


### true

存储 true。

    +--------+
    |  0xc3  |
    +--------+


### false

存储 false。

    +--------+
    |  0xc2  |
    +--------+


## 浮点数


### float

在 5 个字节中存储一个大端格式的 IEEE 754 单精度浮点数。

    +--------+--------+--------+--------+--------+
    |  0xca  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+
    => big-endian IEEE 754 single precision floating point number XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX


### double

在 9 个字节中存储一个大端格式的 IEEE 754 双精度浮点数。

    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    |  0xcb  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|
    +--------+--------+--------+--------+--------+--------+--------+--------+--------+
    => big-endian IEEE 754 single precision floating point number XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX


## 原始字节


### fix raw

存储一个原始字节，最多 31 字节。

    +--------+--------
    |101XXXXX|...N bytes
    +--------+--------
    => 000XXXXXX (=N) bytes of raw bytes.


### raw 16

存储一个原始字节，最多 (2^16)-1 字节。长度存储在一个 16-bit 大端格式的无符号整数中。

    +--------+--------+--------+--------
    |  0xda  |XXXXXXXX|XXXXXXXX|...N bytes
    +--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX (=N) bytes of raw bytes.


### raw 32

存储原始字节，最多 (2^31)-1 字节。长度存储在一个 32-bit 大端格式的无符号整数中。

    +--------+--------+--------+--------+--------+--------
    |  0xdb  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|...N bytes
    +--------+--------+--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX (=N) bytes of raw bytes.



## 数组


### fix array

存储一个数组，最多 15 个元素。

    +--------+--------
    |1001XXXX|...N objects
    +--------+--------
    => 0000XXXX (=N) elements array.


### array 16

存储一个数组，最多 (2^16-1) 个元素。元素个数存储在一个 16-bit 大端格式的无符号整数中。

    +--------+--------+--------+--------
    |  0xdc  |XXXXXXXX|XXXXXXXX|...N objects
    +--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX (=N) elements array.


### array 32

存储一个数组，最多 (2^32-1) 个元素。元素个数存储在一个 32-bit 大端格式的无符号整数中。

    +--------+--------+--------+--------+--------+--------
    |  0xdd  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|...N objects
    +--------+--------+--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX (=N) elements array.


## Map


### fix map

存储一个 map，最多 15 个元素。

    +--------+--------
    |1000XXXX|...N*2 objects
    +--------+--------
    => 0000XXXX (=N) elements map
       其中奇数元素是 key，紧接着下一个元素是 该 key 的 值。


### map 16

存储一个 map，最多 (2^16)-1 个元素。元素个数存在一个 16-bit 大端格式的无符号整数中

    +--------+--------+--------+--------
    |  0xde  |XXXXXXXX|XXXXXXXX|...N*2 objects
    +--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX (=N) elements map
       其中奇数元素是 key，紧接着下一个元素是 该 key 的 值。


### map 32

存储一个 map，最多 (2^32)-1 个元素。元素个数存在一个 32-bit 大端格式的无符号整数中

    +--------+--------+--------+--------+--------+--------
    |  0xdf  |XXXXXXXX|XXXXXXXX|XXXXXXXX|XXXXXXXX|...N*2 objects
    +--------+--------+--------+--------+--------+--------
    => XXXXXXXX_XXXXXXXX_XXXXXXXX_XXXXXXXX (=N) elements map
       其中奇数元素是 key，紧接着下一个元素是 该 key 的 值。



## 类型图表

<table>
<tr><th>Type</th><th>Binary</th><th>Hex</th></tr>
<tr><td>Positive FixNum</td><td>0xxxxxxx</td><td>0x00 - 0x7f</td></tr>
<tr><td>FixMap</td><td>1000xxxx</td><td>0x80 - 0x8f</td></tr>
<tr><td>FixArray</td><td>1001xxxx</td><td>0x90 - 0x9f</td></tr>
<tr><td>FixRaw</td><td>101xxxxx</td><td>0xa0 - 0xbf</td></tr>
<tr><td>nil</td><td>11000000</td><td>0xc0</td></tr>
<tr><td>_reserved_</td><td>11000001</td><td>0xc1</td></tr>
<tr><td>false</td><td>11000010</td><td>0xc2</td></tr>
<tr><td>true</td><td>11000011</td><td>0xc3</td></tr>
<tr><td>_reserved_</td><td>11000100</td><td>0xc4</td></tr>
<tr><td>_reserved_</td><td>11000101</td><td>0xc5</td></tr>
<tr><td>_reserved_</td><td>11000110</td><td>0xc6</td></tr>
<tr><td>_reserved_</td><td>11000111</td><td>0xc7</td></tr>
<tr><td>_reserved_</td><td>11001000</td><td>0xc8</td></tr>
<tr><td>_reserved_</td><td>11001001</td><td>0xc9</td></tr>
<tr><td>float</td><td>11001010</td><td>0xca</td></tr>
<tr><td>double</td><td>11001011</td><td>0xcb</td></tr>
<tr><td>uint 8</td><td>11001100</td><td>0xcc</td></tr>
<tr><td>uint 16</td><td>11001101</td><td>0xcd</td></tr>
<tr><td>uint 32</td><td>11001110</td><td>0xce</td></tr>
<tr><td>uint 64</td><td>11001111</td><td>0xcf</td></tr>
<tr><td>int 8</td><td>11010000</td><td>0xd0</td></tr>
<tr><td>int 16</td><td>11010001</td><td>0xd1</td></tr>
<tr><td>int 32</td><td>11010010</td><td>0xd2</td></tr>
<tr><td>int 64</td><td>11010011</td><td>0xd3</td></tr>
<tr><td>_reserved_</td><td>11010100</td><td>0xd4</td></tr>
<tr><td>_reserved_</td><td>11010101</td><td>0xd5</td></tr>
<tr><td>_reserved_</td><td>11010110</td><td>0xd6</td></tr>
<tr><td>_reserved_</td><td>11010111</td><td>0xd7</td></tr>
<tr><td>_reserved_</td><td>11011000</td><td>0xd8</td></tr>
<tr><td>_reserved_</td><td>11011001</td><td>0xd9</td></tr>
<tr><td>raw 16</td><td>11011010</td><td>0xda</td></tr>
<tr><td>raw 32</td><td>11011011</td><td>0xdb</td></tr>
<tr><td>array 16</td><td>11011100</td><td>0xdc</td></tr>
<tr><td>array 32</td><td>11011101</td><td>0xdd</td></tr>
<tr><td>map 16</td><td>11011110</td><td>0xde</td></tr>
<tr><td>map 32</td><td>11011111</td><td>0xdf</td></tr>
<tr><td>Negative FixNum</td><td>111xxxxx</td><td>0xe0 - 0xff</td></tr>


