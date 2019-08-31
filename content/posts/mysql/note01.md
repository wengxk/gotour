---
title: "MySQL Language Structure Basis"
date: 2019-08-20
type:
- post
- posts
categories:
- mysql
---

# 语言结构基础

## 1. 基本数据类型

### 1.1 字符串

在MySQL中，字符串类型主要包括 `CHAR`, `VARCHAR`, `BINARY`, `VARBINARY`, `BLOB`, `TEXT`, `ENUM`, 和 `SET`。

#### 1.1.1 字符串字面量

字符串是指一个被单引号(')或双引号(")包裹的字节或字符序列。在MySQL中，如果启用了 `ANSI_QUOTES` SQL MODE，那么字符串就仅能使用单引号包裹。

`'a string'`

`"another string"` (invalid if ANSI_QUOTES sql_mode is enabled)

对于连续相邻的字符串，MySQL会自动组合为单个字符串。

`mysql> SELECT 'strings' 'placed' 'adjacently' a  WHERE '1' '1' = '11';`

|a|
|:---|
|stringsplacedadjacently|

#### 1.1.2 CHAR & VARCHAR

CHAR 和 VARCHAR 相似，区别在于存储与检索。

相同点：`CHAR(n)` 和 `VARCHAR(n)` 在定义数据类型时，这里的n指的都是字符长度，而不是字节长度。

不同点：

`CHAR(n)` 表示固定长度n，且要求n为0-255。在存储时，如果用户实际赋值的长度没有达到n，MySQL会在字符后填补空格(0x20)，
在查询时，MySQL引擎会清除掉末尾空格后返回给用户。

> 在进行字符串比较时，末尾空格的处理情况与Collation设置有关

`VARCHAR(n)` 表示可变最大字符长度n。

1. 存储长度不够时既不右侧填充空格，查询时也不会进行右部清除空格。
2. 实际存储时，会添加数据字节长度的前导，占用一个字节或者两个字节。
   不超过255字节，前导占用一字节，超出255字节，前导占用两个字节。
3. 实际情况下按照InnoDB数据存储要求，最多存储65536字节数据（多字段共享最大字节要求，且因其他数据行的存储细节，导致实际多行共享最大存储字节数小于65536）。

> 如果字符比较超长，`VARCHAR` 可能不够用的情况下可以使用 `TEXT` 存储。

示例：

| 值 | CHAR(4) | 存储要求 | VARCHAR(4)| 存储要求 |
|:---|:---|:---|:---|:---|
|''|'&nbsp;&nbsp;&nbsp;&nbsp;'|4 bytes|''|1 bytes|
|'ab'|'ab&nbsp;&nbsp;'|4 bytes|'ab'|3 bytes|
|'abcd'|'abcd'|4 bytes|'abcd'|5 bytes|

#### 1.1.3 BINARY & VARBINARY

`BINARY` 和 `VARBINARY` 相似于 `CHAR` 和 `VARCHAR` ，不同之处在于前者表示字节型字符串，
而后者表示字符型字符串，所以对于 `BINARY` 和 `VARBINARY`，它们的Charset和Collation都是binary，即二进制数值。

`BINARY(n)` 和 `VARBINARY(n)` 中的n表示存储的字节长度，而非 `CHAR(n)` 和 `VARCHAR(n)` 的字符长度。

`BINARY(n)` 在存储时，若需要存储的数据字节长度小于n，则会以`'\0'`填充剩余字节；查询时，按最终存储数据返回，不会过滤右侧`'\0'`。

示例:

```SQL
CREATE TABLE t(a BINARY(10),b VARBINARY(10));
INSERT INTO t(a,b) VALUES('a','a'),('你','你');
```

`mysql> SELECT HEX(a),HEX(b),CAST(a AS CHAR),CAST(b AS CHAR) FROM t;`

| HEX(a) | HEX(b) | CAST(a AS CHAR) | CAST(b AS CHAR)|
|:---|:---|:---|:---|
|61000000000000000000|61|a|a|
|E4BDA000000000000000|E4BDA0|你|你|

可以验证a的ASCII码十六进制为61，在[https://unicode-table.com/en/4F60/](https://unicode-table.com/en/4F60/)
可以查询到汉字“你”的utf8编码十六进制值为E4 BD A0。

#### 1.1.4 BLOB & TEXT

`BLOB` 为二进制大文件对象，`TEXT` 为字符大文件对象。每个都细分为四个类型。

| 类型 | 存储要求 |
|:---|:---|
|TINYBLOB, TINYTEXT|L+1 bytes, L< 2^8|
|BLOB, TEXT|L+2 bytes, L< 2^16|
|MEDIUMBLOB,  MEDIUMTEXT|L+3 bytes, L< 2^24|
|LONGBLOB, LONGTEXT|L+4 bytes, L< 2^32|

#### 1.1.5 ENUM & SET

MySQL不支持传统数据库的CHECK约束，但是可以通过使用 `ENUM` 和 `SET` 数据类型及启用严格模式的sql_mode来实现（另一种实现方式是使用触发器）。

`ENUM` 代表了单选型的字符串类型，`ENUM('options1','options2','options3'...,'optionsN')`;   
`SET` 代表了多选型的字符串类型，`SET('options1','options2','options3'...,'optionsN')`。

`ENUM` 和 `SET` 中的每个选项都有选项索引的概念。

`ENUM('options1','options2','options3'...,'optionsN')`

| 选项 | 索引值 | 备注 |
|:---|:---|:---|
|NULL,|NULL|如果列可为NULL|
|''|0|如果没有启用严格模式，且数据不符合选项的任何一个|
|options1|1|
|options2|2|
|options3|3|
|optionsN|N|

`SET('options1','options2','options3'...,'optionsN')`

| 选项 | 索引值 | 备注 |
|:---|:---|:---|
|NULL,|NULL|如果列可为NULL|
|options1|1|
|options2|2|
|options3|4|
|optionsN|2^(N-1)|

示例：

```SQL
CREATE TABLE t
 (
	 a ENUM('Sun.','Mon.','Tues.','Wed.','Thur.','Fri.','Sat.') NOT NULL,
	 b SET('Sun.','Mon.','Tues.','Wed.','Thur.','Fri.','Sat.') NOT NULL
 );
 INSERT INTO t(a,b) VALUES('Sun.','Sun.,Mon.'),(6,7);
```

`mysql> SELECT * FROM t;`

| a | b |
|:---|:---|
|Sun.|Sun.,Mon.|
|Fri.|Sun.,Mon.,Tues.|

> 因为 `ENUM` 和 `SET` 数据类型都有选项索引的概念，所以要避免以1，2，3或'1'，'2'，'3'这种数字作为选项值

### 1.2 数值

MySQL支持所有标准SQL数值类型，包括定点型和浮点型数值。

#### 1.2.1 BIT

`BIT` 类型存储的是二进制数据类型， `BIT(M)` 存储长度为M的二进制数据，1 <= M <= 64。

二进制类型的字面量表示：

| 字面量 | 十进值 |
|:---|:---|
|b'101'|5|
|B'101'|5|
|0b101|5|
|0B101|非法|
|b101|非法|
|B101|非法|

十六进制的字面量表示：

| 字面量 | 十进值 |
|:---|:---|
|X'10aF'|4271|
|x'10AF'|4271|
|0x10AF|4271|
|0X10AF|非法|
|x10AF|非法|
|X10AF|非法|

#### 1.2.2 INTEGER

整型具体分为以下几种数据类型。

|类型|存储字节|有符号最小值|无符号最小值|有符号最大值|无符号最大值|
|:---|:---|:---|:---|:---|:---|
|TINYINT|1|-128|0|127|255|
|SMALLINT|2|-32,768|0|32,767|65,535|
|MEDIUMINT|3|-8,388,608|0|8,388,607|16,777,215|
|INT|4|-2,147,483,648|0|2,147,483,647|4,294,967,295|
|BIGINT|8|-2^63|0|2^63 - 1 |2^64 - 1|

#### 1.2.3 FIXED-POINT

定点型包括 `DECIMAL` 和 `NUMERIC`，对 `DECIMAL` 的规则适用于 `NUMERIC`。

`DECIMAL(M,D)`，M代表数据总长度，整数位加上小数位，P代表小数位长度。1 <= M <= 65，默认10，0 <= P <= 30，默认0。

`DECIMAL(10,0)`, `DECIMAL(10)`, `DECIMAL` 三者等价。

```SQL
CREATE TABLE t(a DECIMAL(10,2));
INSERT INTO t(a) VALUES(10000); -- 10000.00
INSERT INTO t(a) VALUES(11111234); -- 11111234.00
INSERT INTO t(a) VALUES(50001.2394); -- 50001.24
INSERT INTO t(a) VALUES(111112345); -- error, out of range value
```

MySQL将 `DECIMAL` 的整数和小数部分分开存储，对于每部分，每9位（10进制为基准）占用4字节，不足9位的占用字节数如下：

| 剩余字节数| 字节 |
|:---|:---|
|1-2|1|
|3-4|2|
|5-6|3|
|7-8|4|

例如，DECIMAL(16,6)，整数部分10位，小数部分6位，则共占用 4 + 1 + 3 共8字节。

#### 1.2.4 FLOATING-POINT

浮点数保存的是近似值。

|类型|存储要求|
|:---|:---|
|FLOAT|4 bytes|
|DOUBLE|8 bytes|

形如 `FLOAT(M,D)` 和 `DOUBLE(M,D)` 的类型，尽量少用，在后期的MySQL中可能会逐渐废弃和移除。

#### 1.2.5 NUMERIC TYPE ATTRIBUTES

NUMERIC TYPE ATTRIBUTES 指的是可以在定义整型时限定显示宽度即长度。

```SQL
CREATE TABLE t(a INT(10), b SMALLINT(10));
INSERT INTO t(a,b) VALUES(999999999,32767); -- successs
INSERT INTO t(a) VALUES(11111111111); -- error, out of range value
INSERT INTO t(b) VALUES(32768); -- error, out of range value
```

### 1.3 日期和时间

#### 1.3.1 MySQL中的时区

在说明MySQL日期和时间前，需要先了解下世界时间和时区的基本知识，见 [GMT: Greenwich Mean Time 格林尼治标准时间](https://time.artjoey.com/cn/)

摘抄一段原文：

1. 是指位于英国伦敦郊区的皇家格林尼治天文台的标准时间，因为本初子午线被定义在通过那裡的经线。
2. 理论上来说，格林尼治标准时间的正午是指当太阳横穿格林尼治子午线时（也就是在格林尼治上空最高点时）的时间。
3. 由于地球在它的椭圆轨道里的运动速度不均匀，这个时刻可能和实际的太阳时相差16分钟。
地球每天的自转是有些不规则的，而且正在缓慢减速。所以，格林尼治时间已经不再被作为标准时间使用。
4. 自1924年2月5日开始，格林尼治天文台每隔一小时会向全世界发放调时信息。
5. 现在的标准时间 - 协调世界时（UTC: Coordinated Universal Time）- 由原子钟提供。

即北京时间 = GMT+8 = UTC+8。

**MySQL会将 `TIMESTAMP` 类型的数据转换成UTC时间进行存储，在查询时会根据当前会话的时区返回对应的时间结果。**

查询当前MySQL服务器的时区设置：

`mysql> SELECT @@GLOBAL.time_zone,@@SESSION.time_zone;`

|@@GLOBAL.time_zone|@@SESSION.time_zone|
|:---|:---|
|SYSTEM|SYSTEM|

查询当前时间：

`mysql> SELECT NOW();`

|NOW()|
|:---|
|2019-08-25 16:56:34|

然后我们再设置下当前会话的时区为东九区：

`mysql> SET SESSION time_zone = '+9:00';`

再次查询服务器的时区设置：

`mysql> SELECT @@GLOBAL.time_zone,@@SESSION.time_zone;`

|@@GLOBAL.time_zone|@@SESSION.time_zone|
|:---|:---|
|SYSTEM|+09:00|

查询当前时间：

`mysql> SELECT NOW();`

|NOW()|
|:---|
|2019-08-25 17:59:23|

#### 1.3.2 日期和时间类型

MySQL支持五种和日期时间有关的类型。

|类型|格式|存储要求|最小值|最大值|零值|
|:---|:---|:---|:---|:---|:---|
|DATE|YYYY-MM-DD|3 bytes|1000-01-01|9999-12-31|0000-00-00|
|TIME|hh:mm:ss (or hhh:mm:ss for large value)|3 bytes|-838:59:59|838:59:59|00:00:00|
|DATETIME|YYYY-MM-DD hh:mm:ss|8 bytes|1000-01-0100:00:00|9999-12-31 23:59:59|0000-00-00 00:00:00|
|TIMESTAMP|YYYY-MM-DD hh:mm:ss|4 bytes|1970-01-01 00:00:01|2038-01-19 03:14:07|0000-00-00 00:00:00|
|YEAR|YYYY|1 bytes|1901|2155|0000|

> mysql日期和时间类型的零值和非法值受sql_mode中的 `NO_ZERO_DATE` 和  `ALLOW_INVALID_DATES` 控制。

另外，在MySQL5.6.4及以后版本中， `TIME`， `DATETIME`， `TIMESTAMP` 存储要求有变化。

|类型|存储要求|
|:---|:---|
|TIME|3 bytes + fractional seconds storage|
|DATETIME|4 bytes + fractional seconds storage|
|TIMESTAMP|5 bytes + fractional seconds storage|

|Fractional Seconds Precision|Storage Required|
|:---|:---|
|0|0 bytes|
|1-2|1 bytes|
|3-4|2 bytes|
|5-6|3 bytes|

例如： `TIME(0)`， `TIME(2)`， `TIME(4)`， `TIME(6)` 分别占用 3，4，5，6字节。

#### 1.3.3 日期和时间的格式化

见 [DATE_FORMAT](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-functions.html#function_date-format)

记一组常用格式：

`mysql> SELECT DATE_FORMAT(NOW(),'%Y-%m-%d %H:%i:%s.%f') cur;`

|cur|
|:---|
|2019-08-25 18:14:20.000000|

## 2. 字符集和校对规则

### 2.1 相关概念

我们需要了解基本的相关性概念。

目前的计算机系统是基于二进制系统运行的，所有数据以二进制的补码形式存储。在计算机早期时，一套 `ASCII` 码足够满足使用需求，
但是随着计算机技术和社会水平的发展，越来越多的人及机构开始接入和使用计算机，
这套基于拉丁字母的 `ASCII` 码就无法满足世界各种语言或字符的表示，于是出现了 [`Unicode`](http://www.unicode.org)。

`Unicode` 的出现解决了世界上各种符号的无法统一问题，
Unicode为世界上的任一字符都定义了一个唯一的编号，即 `Unicode` 码，并且兼容之前的 `ASCII` 码。

单个 `Unicode` 码很容易识别和读取，但是多个呢？以什么做限定，是特殊字符做分隔符还是定长表示？
在此期间有过各种编码格式，但是随着互联网的普及，`UTF-8` 编码已经成为事实标准。
`UTF-8` 最大的特点就是它是一种变长的编码格式，可以根据字符的实际长度选择使用1-4个字节来表示。
那 `UTF-8` 到底具有怎样的编码规则呢？

See

- [http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
- [https://www.fileformat.info/info/unicode/utf8.htm](https://www.fileformat.info/info/unicode/utf8.htm)

### 2.2 MySQL中的字符集编码和校对规则

简单的说，字符集指的就是各种符号及其唯一对应的数字编号，校对规则指的是两个符号的排序规则。
MySQL支持多种字符集和校对规则，支持在启动服务器、创建数据库、创建表、创建列及字符串字面量时指定字符集和校对规则。

查看字符集

`mysql> SHOW CHARSET;`

查看uft8字符集有哪些校对规则

`mysql> SHOW COLLATION WHERE CHARSET LIKE 'utf8%';`

如何指定字符集和校对规则的语法可以自己查阅相关资料，这里主要说明 `COLLATION` 的 `pad_attribute` 属性。
`pad_attribute` 指的是否过滤字符串末尾空格，`NO PAD` 不过滤，`PAD SPACE` 过滤。

查看当前会话的校对规则

`mysql> SHOW  VARIABLES LIKE '%collation%';`

|Variable_name|Value|
|:---|:---|
|collation_connection|utf8mb4_0900_ai_ci|
|collation_database|utf8mb4_general_ci|
|collation_server|utf8mb4_0900_ai_ci|
|default_collation_for_utf8mb4|utf8mb4_0900_ai_ci|

上述 `utf8mb4_0900_ai_ci` 是因为在创建数据库时指定了该校对规则，其 `pad_attribute` 为 `PAD SPACE`，
其他的皆为默认校对规则，且 `pad_attribute` 为 `NO PAD`。

如下语句查询返回empty set

`mysql> SELECT 1 WHERE  'a'   = 'a ';`

修改后，返回1

`mysql> SELECT 1 WHERE  'a'   = 'a '  COLLATE utf8mb4_general_ci;`

> **总结**：`CHARSET` 和 `COLLATION` 这两个设置很重要，但是我们并不需要花太多时间在上面，
> 一般情况下只需要使用默认的uft8（如果有utf8mb4那就用utf8mb4）即可。
