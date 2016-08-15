---
title: 理解JVM中class文件结构--概念部分
date: 2016-05-22 13:38:49
tags:
- jvm
- java
categories: jvm

---
#### 什么是class文件
java源代码（后缀名是.java）经过编译器编译之后会生成class文件（后缀名是.class），每一个 Class 文件都对应着唯一一个类或接口的定义信息，但是相对地， 类或接口并不一定都得定义在文件里，我们只是通俗地将任意一个有效的类或接口所应当满足的格式称为“ Class 文件格式”， 即使它不一定以磁盘文件的形式存在。

每个 Class 文件都是由 8 字节为单位的字节流组成，所有的 16 位、 32 位和 64 位长度的数据将被构造成 2 个、 4 个和 8 个 8 字节单位来表示。多字节数据项总是按照 [Big-Endian](https://en.wikipedia.org/wiki/Endianness)的顺序进行存储。
JVM规范还定义了一组私有数据类型来表示 Class 文件的内容，它们包括 `u1， u2 和 u4`，分别代表了 `1、 2 和 4 `个字节的无符号。

为了避免与类的字段、 类的实例等概念产生混淆， 在此把用于描述类结构格式的内容定义为`项`（ Item）。在 Class 文件中，各项按照`严格顺序连续存放`的， 它们之间没有任何填充或对齐作为各项间的分隔符号。

`表`（ Table） 是由任意数量的可变长度的`项`组成，用于表示Class 文件内容的一系列复合结构。

#### class文件结构
每一个 Class 文件对应于一个如下所示的 ClassFile 结构体：

ClassFile :
    `u4 magic;`
    **魔数值固定为 0xCAFEBABE,不会改变，唯一作用是确定这个文件是否为一个能被虚拟机所接受的 Class 文件。**
    `u2 minor_version;`**副版本号**
    `u2 major_version;`**主版本号**
    **主副版本号共同构成了 Class 文件的格式版本号。譬如某个 Class 文件的主版本号为 M，副版本号为 m，那么这个Class 文件的格式版本号就确定为 M.m。一个 Java 虚拟机实例只能支持特定范围内的主版本号。**
    `u2 constant_pool_count;`
    **常量池计数器，constant_pool_count 的值等于 constant_pool 表中的成员数加 1。constant_pool 表的索引值只有在大于 0 且小于 constant_pool_count 时才会被认为是有效的。（0表示不引用常量池的任一项）**
    `cp_info constant_pool[constant_pool_count-1];`
    **常量池，constant_pool 是一种表结构， 它包含 Class 文件结构及其子结构中引用的所有字符串常量、 类或接口名、字段名和其它常量。 常量池中的每一项都具备相同的格式特征——第一个字节作为类型标记用于识别该项是哪种类型的常量，  称为“ tagbyte”。常量池的索引范围是 1 至 constant_pool_count−1。（索引1表示第一个常量项）具体结构信息可查看我的另
    一篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)对应tag。**
    `u2 access_flags;`
    **访问标志， access_flags 是一种掩码标志， 用于表示某个类或者接口的访问权限及基础属性。具体结构信息可查看我的另一篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)对应tag。**
    `u2 this_class;`
    **类索引， this_class 的值必须是对 constant_pool 表中项目的一个有效索引值。constant_pool 表在这个索引处的项必须为 CONSTANT_Class_info 类型常量。**
    `u2 super_class;`
    **父类索引，对于类来说， super_class 的值必须为 0 或者是对 constant_pool 表中项目的一个有效索引值。 如果它的值不为 0，那 constant_pool 表在这个索引处的项必须为 CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类的直接父类。**
    `u2 interfaces_count;`
    **接口计数器， interfaces_count 的值表示当前类或接口的直接父接口数量。**
    `u2 interfaces[interfaces_count];`
    **接口表， interfaces[]数组中的每个成员的值必须是一个对 constant_pool 表中项目的一个有效索引值， 它的长度为 interfaces_count。每个成员 interfaces[i] 必须为 CONSTANT_Class_info 类型常量,成员所表示的接口顺序和对应的源代码中给定的接口顺序（ 从左至右）
    一样。**
    `u2 fields_count;`
    **字段计数器， fields_count 的值表示当前 Class 文件 fields[]数组的成员个数。**
    `field_info fields[fields_count];`
    **字段表， fields[]数组中的每个成员都必须是一个 fields_info 结构的数据项，用于表示当前类或接口中某个字段的完整描述。fields[]数组描述当前类或接口声明的所有字段，但*不包括*从父类或父接口继承的部分。具体结构信息可查看我的另一篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)对应tag。**
    `u2 methods_count;`
    **方法计数器， methods_count 的值表示当前 Class 文件 methods[]数组的成员个数。**
    `method_info methods[methods_count];`
    **方法表， methods[]数组中的每个成员都必须是一个 method_info 结构的数据项，用于表示当前类或接口中某个方法的完整描述。methods[]数组只描述当前类或接口中声明的方法，`不包括`从父类或父接口继承的方法。具体结构信息可查看我的另一篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)对应tag。**
    `u2 attributes_count;`
    **属性计数器， attributes_count 的值表示当前 Class 文件 attributes 表的成员个数。attributes 表中每一项都是一个attribute_info 结构的数据项。**
    `attribute_info attributes[attributes_count];`
    **属性表，和字段表、方法表中属性表中的内容*有所区别*。（某些属性只能用于class,某些属性只能用于字段或者方法，例如ConstantValue 属性用于字段，Code 属性用于方法）具体结构信息可查看我的另一篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)对应tag。**

`以下是JVM规范SE7中的原文：`
JVM规范里， Class 文件结构中的 attributes 表的项包括下列定义的属性：InnerClasses、EnclosingMethod、Synthetic 、Signature、SourceFile，SourceDebugExtension、 Deprecated、 RuntimeVisibleAnnotations、 RuntimeInvisibleAnnotations以及BootstrapMethods属性。对于支持 Class 文件格式版本号为 49.0 或更高的 Java虚拟机实现，必须正确识别并读取 attributes表中的Signature、 RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations属性。
对于支持 Class 文件格式版本号为 51.0 或更高的Java 虚拟机实现，必须正确识别并读取 attributes 表中的BootstrapMethods属性。
本规范要求任一 Java 虚拟机实现可以自动忽略Class文件的 attributes 表中的若干（ 甚至全部） 它不可识别的属性项。任何本规范未定义的属性不能影响 Class 文件的语义，只能提供附加的描述信息。
`概括一下：虚拟机可以忽视某些属性，不同版本的虚拟机实例中所需的属性有所区别，你也可以实现自己的属性，但是只能作为附加信息，不能对原语义做出修改。`

以上结构便是整个class文件，建议和我的另两篇博文[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)以及[《理解JVM中class文件结构--实例讲解》](/2016/05/23/java-classfile-structure-example/)结合的来看。
- 参考《JVM规范SE7》

<font color= Darkorange>尊重他人劳动,转载请在正文明显处注明原文地址。</font>

