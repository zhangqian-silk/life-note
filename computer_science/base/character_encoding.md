# 字符编码

## 技术背景

在计算机中，所有的数据均以二进制方式（0和1）进行存储，想要正确表示自然语言中常用的字符，例如数字 0、1、2，字母 a、b、c，符号 {}、()、[]，以及中文字符等等，则需要指定具体用哪些二进制数字来表示具体的字符，这种映射关系叫做编码。

为了在不同的环境下实现通信，则必须遵循统一的字符编码规范。

## ASICC

> [wiki/ASCII](https://zh.wikipedia.org/wiki/ASCII)

ASCII（American Standard Code for Information Interchange，美国信息交换标准代码）是基于拉丁字母的一套电脑字符编码标准，现已被国际标准化组织（International Organization for Standardization，ISO）定为国际标准（ISO/IEC 646）。

ASICC 的编码范围为 0~127(0x00~0x7f)，包含了所有的英文字母(A-Za-z)、所有的数字(0-9)、部分的标点符号((),[],{}...)、部分运算符号(,+,-,*,/...)以及部分控制字符(换行符、回车符、空字符等)。

## Unicode

> [wiki/Unicode](https://zh.wikipedia.org/wiki/Unicode)

Unicode，全称为 Unicode 标准（The Unicode Standard），是为了解决其他字符编码方案的局限而产生，现已成为业界标准，随着通用字符集 ISO/IEC 10646 的发展而不断增修，截至2022年，Unicode 发布了 15.0.0 版本，已有超过14万个字符。

在 Unicode 的历代版本中，逐步扩充了阿拉伯字符、希腊字母、中日韩统一表意文字、藏文、缅文、交通标志、颜文字等等自然语言符号。

## UTF-8

> [wiki/UTF-8](https://zh.wikipedia.org/wiki/UTF-8)

UTF-8（8-bit Unicode Transformation Format）是一种针对 Unicode 的可变长度字符编码，也是一种前缀码。变长的特性，一方面可以节省内存使用，另一方面则可以同时兼容 ASCII 码，目前已是互联网最主要的编码方式。

UTF-8 编码字节含义如下：

- 对于UTF-8编码中的任意字节B，如果B的第一位为0，则B独立的表示一个字符(ASCII码)；
- 如果B的第一位为1，第二位为0，则B为一个多字节字符中的一个字节(非ASCII字符)；
- 如果B的前两位为1，第三位为0，则B为两个字节表示的字符中的第一个字节；
- 如果B的前三位为1，第四位为0，则B为三个字节表示的字符中的第一个字节；
- 如果B的前四位为1，第五位为0，则B为四个字节表示的字符中的第一个字节；
