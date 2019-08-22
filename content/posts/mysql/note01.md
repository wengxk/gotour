---
title: "Language Structure Basis"
date: 2019-08-20
type:
- post
- posts
categories:
- mysql
---

# 语言结构基础

## 基本数据类型

### 字符串

在MySQL中，字符串类型主要包括`CHAR`,`VARCHAR`, `BINARY`, `VARBINARY`, `BLOB`, `TEXT`, `ENUM`, 和 `SET`。

#### 字符串字面量

字符串是指一个被单引号(')或双引号(")包裹的字节或字符序列。在MySQL中，如果启用了 `ANSI_QUOTES` SQL MODE，那么字符串就仅能使用单引号包裹。

`'a string'`

`"another string"` (invalid if ANSI_QUOTES sql_mode is enabled)

对于连续相邻的字符串，MySQL会自动组合为单个字符串。

`mysql> SELECT 'strings' 'placed' 'adjacently' a  WHERE '1' '1' = '11';`

|a|
|:---|
|stringsplacedadjacently|

#### CHAR & VARCHAR

CHAR 和 VARCHAR 相似，区别在于存储与检索。

相同点：`CHAR(n)` 和 `VARCHAR(n)` 在定义数据类型时，这里的n指的都是字符长度，而不是字节长度。

不同点：

`CHAR(n)` 表示固定长度n，且要求n为0-255。在存储时，如果用户实际赋值的长度没有达到n，MySQL会在字符后填补空格(0x20)，在查询时，MySQL引擎会清除掉末尾空格后返回给用户。

> 在进行字符串比较时，末尾空格的处理情况与Collation设置有关

`VARCHAR(n)` 表示可变最大字符长度n。

1. 存储长度不够时既不右侧填充空格，查询时也不会进行右部清除空格。
2. 实际存储时，会添加数据字节长度的前导，占用一个字节或者两个字节。
   不超过255字节，前导占用一字节，超出255字节，前导占用两个字节。
3. 实际情况下按照InnoDB数据存储要求，最多存储65536字节数据（多字段共享最大字节要求，且因其他数据行的存储细节，导致实际多行共享最大存储字节数小于65536）。

> 如果字符比较超长，`VARCHAR`可能不够用的情况下可以使用`TEXT`存储。

示例：

| 值 | CHAR(4) | 存储要求 | VARCHAR(4)| 存储要求 |
|:---|:---|:---|:---|:---|
|''|'&nbsp;&nbsp;&nbsp;&nbsp;'|4 bytes|''|1 bytes|
|'ab'|'ab&nbsp;&nbsp;'|4 bytes|'ab'|3 bytes|
|'abcd'|'abcd'|4 bytes|'abcd'|5 bytes|

#### BINARY & VARBINARY

`BINARY` 和 `VARBINARY` 相似于 `CHAR` 和 `VARCHAR` ，不同之处在于前者表示字节型字符串，而后者表示字符型字符串，所以对于`BINARY` 和 `VARBINARY`，它们的Charset和Collation都是binary，即二进制数值。

`BINARY(n)` 和 `VARBINARY(n)`中的n表示存储的字节长度，而非`CHAR(n)` 和 `VARCHAR(n)`的字符长度。

`BINARY(n)`在存储时，若需要存储的数据字节长度小于n，则会以`'\0'`填充剩余字节；查询时，按最终存储数据返回，不会过滤右侧`'\0'`。

示例:

```SQL
CREATE TABLE t(a BINARY(10),b VARBINARY(10));
INSERT INTO t(a,b) VALUES('a','a'),('你','你');
```

<!-- `mysql> CREATE TABLE t(a BINARY(10),b VARBINARY(10));`

`mysql> INSERT INTO t(a,b) VALUES('a','a'),('你','你');` -->

`mysql> SELECT HEX(a),HEX(b),CAST(a AS CHAR),CAST(b AS CHAR) FROM t;`

| HEX(a) | HEX(b) | CAST(a AS CHAR) | CAST(b AS CHAR)|
|:---|:---|:---|:---|
|61000000000000000000|61|a|a|
|E4BDA000000000000000|E4BDA0|你|你|

可以验证a的ASCII码十六进制为61，在[https://unicode-table.com/en/4F60/](https://unicode-table.com/en/4F60/)可以查询到汉字“你”的utf8编码十六进制值为E4 BD A0。

#### BLOB & TEXT

`BLOB`为二进制大文件对象，`TEXT`为字符大文件对象。每个都细分为四个类型。

| 类型 | 存储要求 |
|:---|:---|
|TINYBLOB, TINYTEXT|L+1 bytes, L< 2^8|
|BLOB, TEXT|L+2 bytes, L< 2^16|
|MEDIUMBLOB,  MEDIUMTEXT|L+3 bytes, L< 2^24|
|LONGBLOB, LONGTEXT|L+4 bytes, L< 2^32|

#### ENUM & SET

MySQL不支持传统数据库的CHECK约束，但是可以通过使用`ENUM`和`SET`数据类型及启用严格模式的sql_mode来实现（另一种实现方式是使用触发器）。

`ENUM`代表了单选型的字符串类型，`ENUM('options1','options2','options3'...,'optionsN')`;   
`SET`代表了多选型的字符串类型，`SET('options1','options2','options3'...,'optionsN')`。

`ENUM`和`SET`中的每个选项都有选项索引的概念。

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

> 因为`ENUM`和`SET`数据类型都有选项索引的概念，所以要避免以1，2，3或'1'，'2'，'3'这种数字作为选项值

## 数值


## 日期和时间

## 其他
