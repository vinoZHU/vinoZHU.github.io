---
title: 理解JVM中class文件结构--实例讲解
date: 2016-05-23 13:25:43
tags:
- jvm
- java
categories: jvm

---
本文通过一个简单的例子来讲解class文件结构,**某些具体结构概念和信息需结合我之前的两篇博文[《理解JVM中class文件结构--概念部分》](/2016/05/22/java-classfile-structure-concept/)，[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)**，建议可以通过上面的两篇博文先了解这方面的一些知识再看本文。
java代码（jdk7）：
```java
package pac1;

/**
 * Created by vino on 2016/5/21.
 */
public class TestClass {
    int testInt;

    public void test(){
        testInt = 1;
    }

}
```

对应class文件(十六进制)：
![](/images/jvm/java-classfile-structure-example-0.PNG)
接下来的顺序和class文件结构顺序一致
#### `u4 magic;`
文件位置`0x0-0x3`为魔数 CA FE BA BE ,固定不变。
#### `u2 minor_version;`
文件位置`0x4-0x5`为副版本号，值为0x0000。
#### `u2 major_version;`
文件位置`0x6-0x7`为主版本号，值为0x0033。
#### `u2 constant_pool_count;`
文件位置`0x8-0x9`为常量池计数器，值为0x0015，十进制为21。
#### 反编译后的常量池信息
此时通过`javap -v` 命令反编译class文件，查看常量池信息：
```java
Constant pool:
   #1 = Methodref          #4.#17         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#18         //  pac1/TestClass.testInt:I
   #3 = Class              #19            //  pac1/TestClass
   #4 = Class              #20            //  java/lang/Object
   #5 = Utf8               testInt
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lpac1/TestClass;
  #14 = Utf8               test
  #15 = Utf8               SourceFile
  #16 = Utf8               TestClass.java
  #17 = NameAndType        #7:#8          //  "<init>":()V
  #18 = NameAndType        #5:#6          //  testInt:I
  #19 = Utf8               pac1/TestClass
  #20 = Utf8               java/lang/Object
```
常量池中一共有20项，这也证明了`u2 constant_pool_count`的值为常量池项目数+1。
在class文件中各项之间`没有任何填充`或`对齐`作为各项间的分隔符号，紧接着的便是常量池项。
#### `cp_info constant_pool[constant_pool_count-1];`

##### 常量池项1
**通过常量池项1来学习常量池**

通过常量池项的通用格式可知，文件位置`0x10`为常量池项的tagbyte,值为0x0A,查看常量池的 tag 项说明可知常量池项1类型为CONSTANT_Methodref。

接着通过查看CONSTANT_Methodref_info 结构具体信息，文件位置`0x11-0x12`为CONSTANT_Methodref的class_index,为常量池项中某一项的索引，类型必须是CONSTANT_Class。值为0x0004,指向常量池项4，通过查看上面反编译后的常量池信息，可以看到常量池项4是CONSTANT_Class类型的。

文件位置`0x13-0x14`为CONSTANT_Methodref的name_and_type_index，同样指向常量池中的一项，类型必须是CONSTANT_NameAndType。值为0x0011,指向常量池项17，通过查看上面反编译后的常量池信息，可以看到常量池项4是CONSTANT_NameAndType类型的。

**在这里把和常量池项1有关的所有常量池项（1-> 4,17 -> 20,7,8）都讲一下**

###### 常量池项4
常量池项4在文件的开始位置是`0x17`，属于常量池项4的tagbyte,值为0x07,是CONSTANT_Class类型的，和反编译后的常量池信息一致。

接着通过查看CONSTANT_Class_info 结构具体信息，文件位置`0x18-0x19`为CONSTANT_Class的 name_index，指向常量池中的某一项，类型必须是CONSTANT_Utf8，表示字符串常量的值。值为0x0014,指向常量池项20，通过查看上面反编译后的常量池信息，可以看到常量池项20是CONSTANT_Utf8类型的。

###### 常量池项20
常量池项20在文件的开始位置是`0xbf`,属于常量池项20的tagbyte,值为0x01,是CONSTANT_Utf8类型的，和反编译后的常量池信息一致。

接着通过查看CONSTANT_Utf8_info 结构具体信息，文件位置`0xc0-0xc1`为CONSTANT_Utf8的length，表示字符串常量的长度，值为0x0010,长度为16个字节。

文件位置`0xc2-0xd1`为具体内容，通过ascii码转换之后的值便是`java/lang/Object`,和反编译后的常量池信息一致。

###### 常量池项17
常量池项17在文件的开始位置是`0xa4`,属于常量池项17的tagbyte,值为0x0c,是CONSTANT_NameAndType类型的，和反编译后的常量池信息一致。

接着通过查看CONSTANT_NameAndType_info 结构具体信息，文件位置`0xa5-0xa6`为CONSTANT_NameAndType的name_index，指向常量池项的某一项，类型必须为CONSTANT_Utf8，表示字符串常量的值。值为0x0007,指向常量池项7。通过查看上面反编译后的常量池信息，可以看到常量池项7是CONSTANT_Utf8类型的。

文件位置`0xa7-0xa8`为CONSTANT_NameAndType的descriptor_index，指向常量池项的某一项，类型必须为CONSTANT_Utf8，表示字符串常量的值。此字符串为方法描述符，具体可参考《JVM规范》。此处的值为0x0008,指向常量池项8。通过查看上面反编译后的常量池信息，可以看到常量池项8是CONSTANT_Utf8类型的。

###### 常量池项7和8
常量池项7和8均为CONSTANT_Utf8类型的,分析方法和常量池项20一样。

###### 常量池项1总结
常量池项1是CONSTANT_Methodref类型，表示一个方法的应用。通过查看上面反编译后的常量池信息，可以发现此方法为java/lang/Object."<init>":()V，但其中涉及到对其他多个常量池项的引用。一个方法是由类名和方法名决定的，所以分别引用了CONSTANT_Class和CONSTANT_NameAndType，其中CONSTANT_NameAndType表示一个方法名。然后CONSTANT_Class和CONSTANT_NameAndType又分别需要字符串信息来描述，所以又引用了CONSTANT_Utf8类型的项。

**其他常量池项可以按照这个方式来分析。**

#### `u2 access_flags`
文件位置`0xd2-0xd3`表示的是access_flags，值为0x0021,通过查看access_flags的取值范围和相应含义表，得知该值的含义是ACC_PUBLIC（可以被包的类外访问。）和ACC_SUPER（当用到 invokespecial 指令时，需要特殊处理的父类方法。）和反编译之后的文件中的`flags: ACC_PUBLIC, ACC_SUPER`一致。其中invokespecial指令调用了`java/lang/Object."<init>":()V`。

#### `u2 this_class`
文件位置`0xd4-0xd5`表示的是u2 this_class，为本类的引用，指向常量池中的某一项，类型必须为CONSTANT_Class。值为0x0003,表示指向常量池项3。通过查看上面反编译后的常量池信息，可以看到常量池项3是CONSTANT_Class类型的,最终值为pac1/TestClass。

#### `u2 super_class`
文件位置`0xd6-0xd7`表示的是u2 super_class，为父类的引用，指向常量池中的某一项，类型必须为CONSTANT_Class。值为0x0004,表示指向常量池项4。通过查看上面反编译后的常量池信息，可以看到常量池项4是CONSTANT_Class类型的,最终值为java/lang/Object,Object是所有类的父类。

#### `u2 interfaces_count`
文件位置`0xd8-0xd9`表示的是u2 interfaces_count，为接口计数器。值为0x0000,表示没有实现任何接口。所以接下来便不会有`u2 interfaces[interfaces_count]`（接口表）的字节信息。

#### `u2 fields_count`
文件位置`0xda-0xdb`表示的是u2 fields_count，为字段计数器。值为0x0001,表示有一个字段。

#### `field_info fields[fields_count]`
接下来便是表示字段信息的字节，文件位置`0xdc-0xdd`表示的是字段的access_flags，值为0x0000。查看字段 access_flags 标记列表及其含义表后发现没有含义是用0x0000表示的。因为源代码中字段前面没有任何修饰符，表示默认访问权限。access_flags 是一种掩码标志，0x0000表示没有一个是命中的。

文件位置`0xde-0xdf`表示的是字段信息的name_index,指向常量池中的某一项，必须是CONSTANT_Utf8类型的，为字段名。值为0x0005,指向常量池项5，通过查看上面反编译后的常量池信息，可以看到常量池项5是CONSTANT_Utf8类型的,最终值为testInt，和源代码的字段名一致。

文件位置`0xe0-0xe1`表示的是字段信息的descriptor_index,指向常量池中的某一项，必须是CONSTANT_Utf8类型的，为字段名。值为0x0006,指向常量池项6，通过查看上面反编译后的常量池信息，可以看到常量池项6是CONSTANT_Utf8类型的,最终值为I,表示int类型。更多基本类型字符解释可查看[《理解JVM中class文件结构--具体结构信息》](/2016/05/22/java-classfile-structure-detail/)中的字段基本类型字符解释表。

文件位置`0xe2-0xe3`表示的是字段信息的attributes_count，此处指为0。如果字段前面用了final或者字段被注解修饰等，则值便不为0，接下来便是各个属性的描述字节。具体属性信息可参看《JVM规范》。

#### `u2 methods_count`
文件位置`0xe4-0xe5`表示的是u2 methods_count，为方法计数器。值为0x0002,表示有2个方法。

#### `method_info methods[methods_count]`
文件位置`0xe6-0xe7`表示的是第一个方法的access_flags,值为0x0001,通过查看方法 access_flags 标记列表及其含义表可知，该方法为ACC_PUBLIC（public，方法可以从包外访问）。

文件位置`0xe8-0xe9`表示的是第一个方法的name_index，指向常量池中的某一项，必须是CONSTANT_Utf8类型的，为方法名。值为0x0007,指向常量池项7，通过查看上面反编译后的常量池信息，可以看到常量池项7是CONSTANT_Utf8类型的,最终值为`<init>`。

文件位置`0xea-0xeb`表示的是第一个方法的descriptor_index，指向常量池中的某一项，必须是CONSTANT_Utf8类型的，为方法名。值为0x0008,指向常量池项8，通过查看上面反编译后的常量池信息，可以看到常量池项8是CONSTANT_Utf8类型的,最终值为`()V`。

文件位置`0xec-0xed`表示的是第一个方法的attributes_count,为方法属性计数器。值为0x0001,表明有一个属性。

接下来的是表示属性信息的字节，`查看属性的通用格式后`，文件位置`0xee-0xef`表示的是第一个方法中属性信息的attribute_name_index，为属性名索引，指向常量池中的某一项，必须是CONSTANT_Utf8类型的。值为0x0009,指向常量池项9，通过查看上面反编译后的常量池信息，可以看到常量池项9是CONSTANT_Utf8类型的,最终值为`Code`。Code属性的属性名固定为“Code”。

文件位置`0xf0-0xf3`表示属性信息的attribute_length，值为0x0000002f,说明当前属性的长度为47，不包括开始的6个字节（attribute_name_index 和 attribute_length所占用的长度）。

文件位置`0xf4-0xf5`表示属性信息的max_stack，为操作数栈的最大深度值，jvm运行时根据该值分配栈帧。此处值为0x0001,说明此方法调用时分配的最大栈帧为1个字节。

文件位置`0xf6-0xf7`表示属性信息的max_locals，为局部变量表最大存储空间，单位是slot。此处值为0x0001,说明此方法局部变量的最大存储空间为1个字节。

文件位置`0xf8-0xfb`表示属性信息的code_length，为具体的字节码长度。此处值为0x00000005,说明此方法具体字节码长度为5个字节。接下来的5个字节的内容就是具体的字节码。

**剩下的一些属性信息可以按照这个方法，对照属性表来分析。**

#### `u2 attributes_count` & `attribute_info attributes[attributes_count]`
u2 attributes_count 和 attribute_info attributes[attributes_count] 和方法里的属性分析原理一样。


- 参考《JVM规范SE7》

<font color= Darkorange>如若觉得本文尚可，欢迎转载交流,转载请在正文明显处注明原文地址，谢谢！</font>