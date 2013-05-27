---
layout: post
title: "unicode in python"
date: 2012-12-19 11:30
comments: true
categories: Unicode Python
---
在python里，你可能会经常看到如下错误：
>UnicodeDecodeError: 'ascii' codec can't decode byte 0xd6 in position 0: ordinal not in range(128)

#### 一、默认编码 defaultencoding
python 在不同的平台上所采用的默认编码是不一样的。可以通过 sys.getdefaultencoding() 来获取。

	import sys
	print sys.getdefaultencoding()

windows 平台上的默认编码为utf8,linux平台上的默认编码是ascii。

**设置默认编码**
	
	import sys
	reload(sys)
	sys.setdefaultencoding("utf-8")

#### 二、字符串string 与 unicode

#####1. utf-8 的字符串
<pre>
	>>>"中文"
	'\xe4\xb8\xad\xe6\x96\x87'

	>>> type("中文")
	&lt;type 'str'&gt;

	>>> len("中文")
	6
</pre>
中文的utf8编码，每个汉字占用3个byte,两个汉字的长度为6.  
python 中的str 字符串只是某种编码的 bytes 字节序列，而非真正意义上的字符序列。

#####2. unicode
<pre>
	>>> u"中文"
	u'\u4e2d\u6587'

	>>> type(u"中文")
	&lt;type 'unicode'&gt;

	>>> len(u"中文")
	2
</pre>
中文的unicode编码，由两个汉字的unicode point 组成，长度为2.

####三、编码转换
#####1. unicode 转换成目标编码
	
	uni_str = u"中文"
	utf8_str = uni_str.encode("utf-8")
	gbk_str = uni_str.encode("gbk")
	gb2312_str = uni_str.encode("gb2312")
	utf16_str = uni_str.encode("utf-16")

中文不能转换成 ascii,所以采用英文。

	uni_str = u"chinese"
	asc_str = uni_str.encode("ascii")

#####2. 其它编码转换成 unicode

	us1 = unicode(utf8_str,"utf-8")	
	us2 = unicode(gbk_str,"gbk")	
	us3 = unicode(gb2312_str,"gb2312")	
	us4 = unicode(utf16_str,"utf-16")	
	us5 = unicode(asc_str,"ascii")	

也可以采用 string.decode的方式，转换成utf8
	
	us1 = "中文".decode("utf-8")
	us2 = gbk_str.decode("gbk")

#####3.判断字符串的编码

- isinstance(s, str) 用来判断是否为一般字符串 
- isinstance(s, unicode) 用来判断是否为unicode 

#### 四、python 文件的编码

代码文件 \# -*- coding=utf-8 -*-或者 #coding=utf-8 

◆ sys.setdefaultencoding("utf-8") 不一定适应于所有的python 版本？  
**解决方案**  
1. Decode earyly  
2. Unicode everywhere  
3. Encode late

#####1. 读写文件

	# coding: UTF-8
	 
	f = open('test.txt')
	s = f.read()
	f.close()
	print type(s) # <type 'str'>
	# 已知是UTF-8编码，解码成unicode
	u = s.decode('UTF-8')

内置的open()方法打开文件时，read()读取的是str，读取后需要使用正确的编码格式进行decode()。
	 
	f = open('test.txt', 'w')
	# 编码成UTF-8编码的str
	s = u.encode('UTF-8')
	f.write(s)
	f.close()

write()写入时，如果参数是unicode，则需要使用你希望写入的编码进行encode()，如果是其他编码格式的str，则需要先用该str的编码进行decode()，转成unicode后再使用写入的编码进行encode()。

#####2. 根据字符串是否unicode编码，来进行解码

	def to_unicode_or_bust(obj, encoding='utf-8'):
	     if isinstance(obj, basestring):
	         if not isinstance(obj, unicode):
	             obj = unicode(obj, encoding)
	     return obj

#####3. 使用codecs 进行文件读写操作

	import codecs
 
	f = codecs.open('test.txt', encoding='UTF-8')
	u = f.read()
	f.close()
	print type(u) # <type 'unicode'>
	 
	f = codecs.open('test.txt', 'a', encoding='UTF-8')
	# 写入unicode
	f.write(u)
	 
	# 写入str，自动进行解码编码操作
	# GBK编码的str
	s = '汉'
	print repr(s) # '\xba\xba'
	# 这里会先将GBK编码的str解码为unicode再编码为UTF-8写入
	f.write(s) 
	f.close()


#####4. BOM文件头

decoding UTF-16 会自动移除BOM，但是UTF-8不会，需要进行 s.decode('utf-8-sig')  
也可以使用chardet.detect() 来检查文件编码。


#### 五、Python3

- 在python3 中 &lt;type 'str'&gt; 是一个unicode 对象。
- separate &lt;type 'bytes'&gt; type
- 所有内建模块支持unicode
- 不再需要 u'text' 的写法
- open() 方法可以像 codecs.open() 一样接受 encoding 编码参数
- 默认编码是UTF-8.



#### 六、API参考

**str(object='')**  
返回以良好格式表示一个对象，对于字符串对象将返回字符串本身。str() 的可读性更好，无法用于eval求值；而 repr(object) 则返回一个可以被 eval() 接受的字符串格式。

unicode(object='')  
unicode(object[, encoding[, errors]])  


**str.decode([encoding[, errors]])**

**str.encode([encoding[, errors]])**


**basestring()  **
是 **str** 和 **unicode** 的superclass,不能被直接调用或实例化，但可以用来判断一个对象object 是否 str或 unicode 的实例。  
isinstance(obj,basestring) 等同于 isinstance(obj,(str,unicode))

**isinstance(object, classinfo)**  
Return true if the object argument is an instance of the classinfo argument, 

**bytearray([source[, encoding[, errors]]])**  
bytearray 会返回一个字节数组。bytearray 类型是一个含有 0<= x < 256 数字的可变序列。  
source 参数：  

- 如果是字符串，bytearray() 相当于 str.encode()
- 如果是整数，会被初始化为长度为整数值，bytes为null的数组
- 如果是个对象并且实现了buffer 接口，一个只读的该对象buffer会被用来初始化该数组
- 如果是 iterable（可迭代的对象），必须是 0<=x< 256 整数的迭代，用来初始化数组
- 如果没有参数，会创建一个长度为0的数组

**chr(i)**   
返回一个包含一个该整数所对应的ascii字符的字符串，例如  chr(97) 返回字符串'a'。这个方法是 ord的逆过程，参数必须在[0..255]内，超过这个范围的会产生异常。

**unichr(i)**
返回该整数值对应的unicode 字符串。

**type(object)**  
返回该object 的类型。也可用isinstance() 测试是否为指定类型的对象.

**type(name, bases, dict)**
返回一个新的类型对象.  
name 对应于 \__name__ 属性； base 对应于 \__base__ 属性； dict 对应于 \__dict__ 属性。  
示例：  X = type('X', (object,), dict(a=1))

####七、编码标准

ISO 8859-1  
　　ISO/IEC 8859-1，又称Latin-1或“西欧语言”，是国际标准化组织内ISO/IEC 8859的第一个8位字符集。

####八、tornado 中的编码处理

tornado 对于http header 的编码：
>HTTP headers are generally ascii (officially they're latin1, but use of non-ascii is rare), so we mostly represent them (and data derived from them) with native strings (note that in python2 if a header contains non-ascii data tornado will decode the latin1 and re-encode as utf8!)

也就是说tornado 会把header 中非utf-8的字符全部解码成latin1 (即ISO-8859-1)，再进行utf-8编码。

为了取到正确包含中文的header值(utf-8编码)，需要先进行utf-8解码，再编码成latin1,最后以utf-8 进行解码。如下：

	value = header_value.decode('utf-8').encode('latin-1').decode('utf-8')


参考资料

- [Unicode In Python, Completely Demystified](http://farmdev.com/talks/unicode/)
- [Python字符编码详解](http://www.cnblogs.com/huxi/archive/2010/12/05/1897271.html)
- [python 编码问题总结](http://hi.baidu.com/ekpumiyvnciouye/item/d6e4773a238e24bd633afffc)
- [python str与bytes之间的转换](http://blog.csdn.net/yatere/article/details/6606316)
- [Guaranteed conversion to unicode or byte string](http://code.activestate.com/recipes/466341-guaranteed-conversion-to-unicode-or-byte-string/)
- [Built-in Functions](http://docs.python.org/2/library/functions.html) - python docs
- [Standard Encodings](http://docs.python.org/2/library/codecs.html#standard-encodings) - python standard-encodings
- [Strings and Unicode](https://github.com/facebook/tornado/wiki/Strings-and-Unicode) - tornado中编码的处理
- [Unicode](http://zh.wikipedia.org/wiki/Unicode) - Wiki
- [UTF-8](http://zh.wikipedia.org/wiki/UTF-8)- Wiki
- [ISO/IEC 8859-1](http://zh.wikipedia.org/wiki/ISO/IEC_8859-1) - Wiki


	
	