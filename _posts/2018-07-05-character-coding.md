---
title: 字符编码
date: 2018-07-05 18:16:30
categories:
- Common
- MySQL
tags:
- MySQL
- Character Coding
---

&emsp;

## 字符、字符集和字符编码

> | 字符 | 基本信息单元，字母、数字和标点等 |
> | 字符集 | 字符集合，例如ASCII、GBK |
> | 字符编码 | 字符集的二进制编码方式 |

那么问题来了，我们常说的ASCII、GBK等都是指字符编码，为什么被列入字符集里了？

实际上对于大部分的字符集来说，一个字符集智慧有一套编码，所以字符集即唯一确定了其编码。

但是凡事总有例外，UNICODE字符集就是一个特例，UNICODE存在多种编码方式，包括UTF-8、UTF-16等。

## 常见字符编码

__ASCII__

```
单字节编码字符集，最高位不使用，常设为0.

0~31及127为控制码，共33个

32~126是字符，共95个

空间占用：1字节
```

__不够用了怎么办 -> LATIN-1（ISO-8859-1）__

```
ASCII扩展字符集，启用最高位。

0~127位与ASCII相同，新开启的128~255收录新字符。

空间占用：1字节
```

__有中文了怎么办 -> GB2312__

```
有中文了怎么办？常用中文字符有6000多个~

GB2312解决了中文问题，包含常用中文字符

0~127位与ASCII相同，高位其他字符取消，采用新规则：

当两个大于127的字符连在一起时，就表示一个汉字，前一个为高字节（0xA1到0xF7），后一个为低字节（0xA1到0xFE），可以组合出7000+的汉字。其中，高字节等于字符区号+0xA0，低字节等于字符所在区中的位置+0xA0。

同时，GB2312也对一些特殊字符、数字和标点等进行了重新编码，重新编码的这一批字符，就是我们常说的“全角”字符，而127位以下的这些字符就被称为“半角”字符。
```

__哎呀，中文那么多不够用啊 -> GBK__

```
基于GB2312进行扩展，不再要求低字节一定是127之后的编码，只要第一个字节是大于127的，那么代表这是一个汉字的开始。

GBK包含了GB2312的所有内容，同时又增加了近20000个新的汉字。


看一个例子：

a机智

采用GBK编码后，以十六进制方式查看，结果为：

61bb fad6 c7

第一个字节小于127，和ASCII编码保持一致，可以直接查表得到61对应a；

之后一共4个字节，分别对应“机智”，查询后匹配bbfa对应”机“，d6c7对应”智”

```

__繁体字  -> BIG5__

```
虽然GBK支持少量繁体中文，但是数量有限，于是就出现了BIG5啦。
```

__ISO表示我坐不住了 -> UNICODE__

```
不同的国家，不同的语言，编码太多，就会出现兼容、转换等问题。这时候，ISO坐不住了。

ISO统一了所有语言、字符和数字等编码，创建了UNICODE（Universal Multiple-Octet Coded Character Set），为了能够同时包含大量的字符，UNICODE使用2个字节来统一表示，对于ASCII中低于127位的字符保持不变，其他全部重新编码。

前面有提到，UNICODE是字符集，并不能代表字符编码，UNICODE拥有多种编码方式，包括为UTF-8、UTF-16。（UCS Transfer Format，UCS是UNICDOE的简称）
```

## 字符与字节

```
不同的编码的字符串，其长度和大小怎么计算？

字符串长度：字符串的实际长度，计算字符长度，strlen结果；
字符串大小：字符串的空间占用大小，需要结合字符编码进行大小计算。
```

## MySQL下的字符编码

show variables like ‘%character%’;

![mysql_coding](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_coding.png?raw=true)

> | character_set_client | 客户端字符编码 |
> | character_set_connection | 网络传输数据的字符编码 |
> | character_set_database | 服务端数据字符编码 |
> | character_set_filesystem | 服务端文件名字符编码 |
> | character_set_results | 服务端返回结果集的字符编码 |
> | character_set_server | 服务端全局字符编码

## MySQL字符集校对规则

相同字符集内，字符比较和排序的规则。

如果查看MySQL中，information_schema的CHARACTER_SETS，可以看到MySQL支持的字符集及其校对规则，例如：

![mysql_collate](https://github.com/Taaang/blog/blob/master/assets/images/post_imgs/img_mysql_collate.png?raw=true)

图中显示了MySQL支持的字符集，以及对应字符集的校对规则、字符集中字符的长度。其中，不同的校对规则，会以_ci、_cs、_bin结尾，分别代表了不同的比对模式，_ci为大小写不敏感，_cs为大小写敏感，_bin为二进制比对。

## MySQL表字符集修改

```
修改表字符集（新数据生效）
	ALTER TABLE  … CHARACTER SET …
修改表字符集（新旧数据生效）
	ALTER TABLE  … CONVERT TO CHARACTER SET …
修改当前会话字符集
	SET NAMES …（client、result、connection）
```  
