---
title: 聊聊java中codepoint和UTF-16相关的一些事
date: 2016-10-07 13:43:06
tags: 
- java
- String
- utf-16
categories: java
---
#### Unicode和UTF-8/UTF-16/UTF-32的关系
Unicode和UTF-8/UTF-16/UTF-32之间就是字符集和编码的关系。字符集的概念实际上包含两个方面，一个是字符的集合，一个是编码方案。字符集定义了它所包含的所有符号，狭义上的字符集并不包含编码方案，它仅仅是定义了属于这个字符集的所有符号。但通常来说，一个字符集并不仅仅定义字符集合，它还为每个符号定义一个二进制编码。当我们提到GB2312或者ASCII的时候，它隐式地指明了编码方案是GB2312或者ASCII，在这些情况下可以认为字符集与编码方案互等。

但是Unicode具有多种编码方案。Unicode字符集规定的标准编码方案是UCS-2（UTF-16），用两个字节表示一个Unicode字符（UTF-16中两个字节的为基本多语言平面字符，4个字节的为辅助平面字符）。而UCS-4(UTF-32）用4个字节表示一个Unicode字符。另外一个常用的Unicode编码方案--UTF-8用1到4个变长字节来表示一个Unicode字符，并可以从一个简单的转换算法从UTF-16直接得到。所以在使用Unicode字符集时有多种编码方案，分别用于合适的场景。

再通俗一点地讲，Unicode字符集就相当于是一本字典，里面记载着所有字符（即图像）以及各自所对应的Unicode码（与具体编码方案无关），UTF-8/UTF-16/UTF-32码就是Unicode码经过相应的公式计算得到的并且实际存储、传输的数据。

#### UTF-16
JVM规范中明确说明了java的char类型使用的编码方案是UTF-16，所以先来了解下UTF-16。

Unicode的编码空间从U+0000到U+10FFFF，共有1112064个码位(code point)可用来映射字符，码位就是字符的数字形式。这部分编码空间可以划分为17个平面（plane）,每个平面包含2^16（65536）个码位。第一个平面称为基本多语言平面（Basic Multilingual Plane, BMP），或称第零平面（Plane 0）。其他平面称为辅助平面（Supplementary Planes）。基本多语言平面内，从U+D800到U+DFFF之间的码位区块是永久保留不映射到Unicode字符。UTF-16就利用保留下来的0xD800-0xDFFF区段的码位来对辅助平面的字符的码位进行编码。

最常用的字符都包含在BMP中，用2个字节表示。辅助平面中的码位，在UTF-16中被编码为一对16比特长的码元，称作代理对（surrogate pair），具体方法是：

1. 将码位减去0x10000,得到的值的范围为20比特长的0~0xFFFFF。
2. 高位的10比特的值（值的范围为0~0x3FF）被加上0xD800得到第一个码元或称作高位代理（high surrogate），值的范围是0xD800~0xDBFF.由于高位代理比低位代理的值要小，所以为了避免混淆使用，Unicode标准现在称高位代理为前导代理（lead surrogates）。
3. 低位的10比特的值（值的范围也是0~0x3FF）被加上0xDC00得到第二个码元或称作低位代理（low surrogate），现在值的范围是0xDC00~0xDFFF.由于低位代理比高位代理的值要大，所以为了避免混淆使用，Unicode标准现在称低位代理为后尾代理（trail surrogates）。

例如U+10437编码:

- 0x10437减去0x10000,结果为0x00437,二进制为0000 0000 0100 0011 0111。
- 分区它的上10位值和下10位值（使用二进制）:0000000001 and 0000110111。
- 添加0xD800到上值，以形成高位：0xD800 + 0x0001 = 0xD801。
- 添加0xDC00到下值，以形成低位：0xDC00 + 0x0037 = 0xDC37。

由于前导代理、后尾代理、BMP中的有效字符的码位，三者互不重叠，搜索时一个字符编码的一部分不可能与另一个字符编码的不同部分相重叠。所以可以通过仅检查一个码元（构成码位的基本单位，2个字节）就可以判定给定字符的下一个字符的起始码元。

#### java中的codepoint相关
对于一个字符串对象，其内容是通过一个char数组存储的。char类型由2个字节存储，这2个字节实际上存储的就是UTF-16编码下的码元。我们使用charAt和length方法的时候，返回的实际上是一个码元和码元的数量，虽然一般情况下没有问题，但是如果这个字符属于辅助平面字符，以上2个方法便无法得到正确的结果。正确的处理方式如下：

```java
int character = aString.codePointAt(i);
int length = aString.codePointCount(0, aString.length())；
```
需要注意codePointAt的返回值，是int而非char,这个值就是Unicode码。

codePointAt方法调用了codePointAtImpl：

```java
static int codePointAtImpl(char[] a, int index, int limit) {
        char c1 = a[index];
        if (isHighSurrogate(c1) && ++index < limit) {
            char c2 = a[index];
            if (isLowSurrogate(c2)) {
                return toCodePoint(c1, c2);
            }
        }
        return c1;
    }
```
isHighSurrogate方法判断下标字符的2个字节是否为UTF-16中的前导代理（0xD800~0xDBFF）：

```java
public static boolean isHighSurrogate(char ch) {
        // Help VM constant-fold; MAX_HIGH_SURROGATE + 1 == MIN_LOW_SURROGATE
        return ch >= MIN_HIGH_SURROGATE && ch < (MAX_HIGH_SURROGATE + 1);
    }
```

```java
public static final char MIN_HIGH_SURROGATE = '\uD800';

public static final char MAX_HIGH_SURROGATE = '\uDBFF';
```

然后++index,isLowSurrogate方法判断下一个字符的2个字节是否为后尾代理（0xDC00~0xDFFF）：

```java
public static boolean isLowSurrogate(char ch) {
        return ch >= MIN_LOW_SURROGATE && ch < (MAX_LOW_SURROGATE + 1);
    }
```

```java
public static final char MIN_LOW_SURROGATE  = '\uDC00';

public static final char MAX_LOW_SURROGATE  = '\uDFFF';
```

toCodePoint方法将这2个码元组装成一个Unicode码：

```java
public static int toCodePoint(char high, char low) {
        // Optimized form of:
        // return ((high - MIN_HIGH_SURROGATE) << 10)
        //         + (low - MIN_LOW_SURROGATE)
        //         + MIN_SUPPLEMENTARY_CODE_POINT;
        return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT
                                       - (MIN_HIGH_SURROGATE << 10)
                                       - MIN_LOW_SURROGATE);
    }
```
 这个过程就是以上将一个辅助平面的Unicode码位转换成2个码元的逆过程。
 
 所以，枚举字符串的正确方法：

```java
for (int i = 0; i < aString.length();) {
	int character = aString.codePointAt(i);
	//如果是辅助平面字符，则i+2
	if (Character.isSupplementaryCodePoint(character)) i += 2;
	else ++i;
}
```
将codePoint转换为char[]可调用Character.toChars方法，然后可进一步转换为字符串：

```java
new String(Character.toChars(codePoint));
```
toChars方法所做的就是以上将Unicode码位转换为2个码元的过程,这2个码元最后就成了字符串对象包含的char数组中的2个元素。

参考：

- 维基百科
- https://www.zhihu.com/question/30945431/answer/50046808

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 
