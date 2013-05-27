---
layout: post
title: "unicode in java"
date: 2012-12-19 11:29
comments: true
categories: Java Unicode
---

####1. **Unicode**  
Unicode（统一码、万国码、单一码、标准万国码）是计算机科学领域里的一项业界标准。它对世界上大部分的文字系统进行了整理、编码，使得电脑可以用更为简化地方式来呈现和处理文字。
  
Unicode依随着通用字符集(UCS)的标准而发展。在表示一个Unicode的字符时，通常会用“U+”然后紧接着一组十六进制的数字来表示这一个字符。

**通用字符集**
通用字符集（Universal Character Set，UCS）是由ISO制定的ISO 10646（或称ISO/IEC 10646）标准所定义的标准字符集。

**编码方式**   

**代码点（code point）**是指与一个编码表中的某个字符对应的代码值。在Unicode标准中，代码点采用16进制书写，并加上前缀U+,例如U+0041就是字母A的代码点。

统一码的编码方式与ISO 10646的通用字符集概念相对应。目前实际应用的统一码版本对应于UCS-2，使用16位的编码空间。也就是每个字符占用2个字节。这样理论上一共最多可以表示216（即65536）个字符。

上述16位统一码字符构成**基本多文种平面**,编码范围从U+000 到 U+FFFF。  

最新（但未实际广泛使用）的统一码版本定义了**16个辅助平面**，两者合起来至少需要占据21位的编码空间，比3字节略少。但事实上辅助平面字符仍然占用4字节编码空间，与UCS-4保持一致。
辅助平面的编码范围从U+10000 到 U+10FFFF。

**Unicode实现方式**  
Unicode的实现方式不同于编码方式。一个字符的Unicode编码是确定的。但是在实际传输过程中，由于不同系统平台的设计不一定一致，以及出于节省空间的目的，对Unicode编码的实现方式有所不同。Unicode的实现方式称为Unicode转换格式（Unicode Transformation Format，简称为UTF）

####2. **UTF-8**  
UTF-8（8-bit Unicode Transformation Format）是一种针对Unicode的可变长度字符编码（定长码），也是一种前缀码。它可以用来表示Unicode标准中的任何字符，且其编码中的第一个字节仍与ASCII兼容，这使得原来处理ASCII字符的软件无须或只须做少部份修改，即可继续使用。因此，它逐渐成为电子邮件、网页及其他存储或传送文字的应用中，优先采用的编码。

UTF-8使用一至四个字节为每个字符编码：   

- 128个US-ASCII字符只需一个字节编码（Unicode范围由U+0000至U+007F）。  
- 带有附加符号的拉丁文、希腊文、西里尔字母、亚美尼亚语、希伯来文、阿拉伯文、叙利亚文及它拿字母则需要二个字节编码（Unicode范围由U+0080至U+07FF）。  
- 其他基本多文种平面（BMP）中的字符（这包含了大部分常用字）使用三个字节编码。  
- 其他极少使用的Unicode 辅助平面的字符使用四字节编码。  

对上述提及的第四种字符而言，UTF-8使用四个字节来编码似乎太耗费资源了。但UTF-8对所有常用的字符都可以用三个字节表示，而且它的另一种选择，**UTF-16编码**，对前述的第四种字符同样需要四个字节来编码，所以要决定UTF-8或UTF-16哪种编码比较有效率，还要视所使用的字符的分布范围而定。


####3. **UTF-16**
UTF-16是Unicode字符集的一种转换方式，即把Unicode的码位转换为16比特长的码元串行，以用于数据存储或传递。UTF是"Unicode/UCS Transformation Format"的首字母缩写，即把Unicode字符转换为某种格式之意。

UTF-16描述  
Unicode的码空间从U+0000到U+10FFFF，共有1,112,064个码位(code point)可用来映射字符. Unicode的码空间可以划分为17个平面(plane)，每个平面包含216(65,536)个码位。每个平面的码位可表示为从U+xx0000到U+xxFFFF, 其中xx表示十六进制值从00<sub>16</sub> 到10<sub>16</sub>，共计17个平面。  

第一个平面成为基本多文种平面（Basic Multilingual Plane, BMP），或称第零平面（Plane 0）。其他平面称为辅助平面(Supplementary Planes)。基本多语言平面内，从U+D800到U+DFFF之间的码位区段是永久保留不映射到字符，因此UTF-16利用保留下来的0xD800-0xDFFF区段的码位来对辅助平面的字符的码位进行编码。

**从U+0000至U+D7FF以及从U+E000至U+FFFF的码位**  
第一个Unicode平面(码位从U+0000至U+FFFF)包含了最常用的字符。该平面被称为基本多语言平面，缩写为BMP. UTF-16与UCS-2编码这个范围内的码位为**单个16比特长的码元**，数值等价于对应的码位. BMP中的这些码位是仅有的码位可以在UCS-2被表示.

**从U+10000到U+10FFFF的码位**  
辅助平面(Supplementary Planes)中的码位，在UTF-16中被编码为一对**16比特长的码元(即32bit,4Bytes)**

####4. **ASCII**   
ASCII（发音： /ˈæski/ ASS-kee[1]，American Standard Code for Information Interchange，美国信息交换标准代码）是基于拉丁字母的一套电脑编码系统。
ASCI至今为止共定义了128个字符；其中33个字符无法显示（这是以现今操作系统为依归，但在DOS模式下可显示出一些诸如笑脸、扑克牌花式等8-bit符号），且这33个字符多数都已是陈废的控制字符。控制字符的用途主要是用来操控已经处理过的文字。在33个字符之外的是95个可显示的字符，包含用键盘敲下空白键所产生的空白字符也算1个可显示字符（显示为空白）。

####**5. ISO/IEC 8859-1**  
ISO 8859-1，正式编号为ISO/IEC 8859-1:1998，又称Latin-1或“西欧语言”，是国际标准化组织内ISO/IEC 8859的第一个8位字符集。它以ASCII为基础，在空置的0xA0-0xFF的范围内，加入96个字母及符号，藉以供使用附加符号的拉丁字母语言使用。曾推出过 ISO 8859-1:1987 版。


#####6. Unicode in Java

char 数据类型（和 Character 对象封装的值）基于原始的 Unicode 规范，将字符定义为固定宽度的 16 位实体。

从 U+0000 到 U+FFFF 的字符集有时也称为*Basic Multilingual Plane (BMP)*。代码点大于 U+FFFF 的字符称为*增补字符*。Java 2 平台在 char 数组以及 String 和 StringBuffer 类中使用**UTF-16**表示形式。在这种表现形式中，增补字符表示为**一对char** 值，第一个值取自高代理项 范围，即 (\uD800-\uDBFF)，第二个值取自低代理项 范围，即 (\uDC00-\uDFFF)。 

**所以，char 值表示 Basic Multilingual Plane (BMP) 代码点，其中包括代理项代码点，或 UTF-16 编码的代码单元。**int 值表示所有 Unicode 代码点，包括增补代码点。int 的 21 个低位（最低有效位）用于表示 Unicode 代码点，并且 11 个高位（最高有效位）必须为零。除非另有指定，否则与增补字符和代理项 char 值有关的行为如下： 

- **只接受一个 char 值的方法无法支持增补字符。**它们将代理项字符范围内的 char 值视为未定义字符。例如，Character.isLetter('\uD840') 返回 false，即使是特定值，如果在字符串的后面跟着任何低代理项值，那么它将表示一个字母。 
- **接受一个 int 值的方法支持所有 Unicode 字符，其中包括增补字符。**例如，Character.isLetter(0x2F81A) 返回 true，因为代码点值表示一个字母（一个 CJK 象形文字）。 

参考资料

- JDK
- [Unicode](http://zh.wikipedia.org/wiki/Unicode) - Wiki
- [UTF-8](http://zh.wikipedia.org/wiki/UTF-8)- Wiki
- [UTF-16](http://zh.wikipedia.org/wiki/UTF-16)- Wiki
- [ISO/IEC 8859-1](http://zh.wikipedia.org/wiki/ISO/IEC_8859-1) - Wiki
- [JAVA中的UTF-16编码](http://www.cnblogs.com/maxupeng/archive/2011/06/22/2087579.html)








