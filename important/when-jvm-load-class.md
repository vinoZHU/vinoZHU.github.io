---
title: 理解JVM加载class原理--class加载的时机
date: 2016-05-19 19:00:58
tags:
- jvm
- java
categories: jvm

---
### java的动态性
Java 语言是一种具有动态性的解释型编程语言，当指定程序运行的时候， Java 虚拟机就将编译生成的 . class 文件按照需求和一定的规则加载进内存，并组织成为一个完整的 Java 应用程序。 Java 语言把每个单独的类 Class 和接口 Implements 编译成单独的一个 . class 文件，这些文件对于 Java 运行环境来说就是一个个可以动态加载的单元。正是因为 Java 的这种特性，我们可以在不重新编译其它代码的情况下，只编译需要修改的单元，并把修改文件编译后的 . class 文件放到 Java 的路径当中， 等到下次该 Java 虚拟机器重新激活时，这个逻辑上的 Java 应用程序就会因为加载了新修改的 .class 文件，自己的功能也做了更新，这就是 Java 的动态性。

### 类的生命周期
 类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括了：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（using）、和卸载（Unloading）七个阶段。其中验证、准备和解析三个部分统称为连接（Linking），这七个阶段的发生顺序如下图所示：
 ![](/images/jvm/when-jvm-load-class-0.PNG)
 如上图所示，加载、验证、准备、初始化和卸载这五个阶段的顺序是确定的，类的加载过程必须按照这个顺序来按部就班地开始，而解析阶段则不一定，它在某些情况下可以在初始化阶段后再开始。

类的生命周期的每一个阶段通常都是互相交叉混合式进行的，通常会在一个阶段执行的过程中调用或激活另外一个阶段。
### 类加载的时机
Java虚拟机规范没有强制性约束在什么时候开始类加载过程，但是对于初始化阶段，虚拟机规范则严格规定了有且只有四种情况必需立即对类进行“初始化”（而加载、验证、准备阶段则必需在此之前开始），这四种情况归类如下：
1.遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。生成这4条指令最常见的Java代码场景是：使用new关键字实例化对象时、读取或者设置一个类的静态字段（被final修饰、已在编译器把结果放入常量池的静态字段除外）时、以及调用一个类的静态方法的时候。
2.使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。
3.当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要触发父类的初始化。
4.当虚拟机启动时，用户需要指定一个执行的主类（包含main()方法的类），虚拟机会先初始化这个类。

对于这四种触发类进行初始化的场景，在java虚拟机规范中限定了“有且只有”这四种场景会触发。这四种场景的行为称为对类的`主动引用`，除此以外的所有引用类的方式都不会触发类的初始化，称为`被动引用`。

### 被动引用

下面通过三个实例来说明被动引用：
#### 示例一
 父类SuperClass.java
 ```java
 package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class SuperClass {
    static{
        System.out.println("SuperClass init!");
    }
    public static int value = 123;
}
```
子类SubClass.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class SubClass extends SuperClass {
    static{
        System.out.println("SubClass init!");
    }
}
```
主类NotInitialization.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class NotInitialization {
    public static void main(String[] args) {
        System.out.println(SubClass.value);
    }
}
```
输出结果
```java
SuperClass init!
123
```
由结果可以看出只输出了“SuperClass init！”，没有输出“SubClass init！”。这是因为对于静态字段，只有直接定义该字段的类才会被初始化，因此当我们通过子类来引用父类中定义的静态字段时，只会触发父类的初始化，而不会触发子类的初始化。

#### 示例二
父类SuperClass.java如上一个示例一样
主类NotInitialization.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class NotInitialization {
  public static void main(String[] args) {  
      SuperClass[] scs = new SuperClass[10];  
  }  
}
```
`输出结果为空`
没有输出“SuperClass init！”说明没有触发类com.chenzhou.classloading.SuperClass的初始化阶段，但是这段代码会触发“[Lcom.chenzhou.classloading.SuperClass”类的初始化阶段。这个类是由虚拟机自动生成的，该创建动作由newarray触发。`从这里可以看出，对象的数组在java里面是一个单独的类型。`

#### 示例三
常量类ConstClass.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class ConstClass {
    static{
        System.out.println("ConstClass init!");
    }

    public static final String HELLOWORLD = "hello world";
}
```
主类NotInitialization.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class NotInitialization {
  public static void main(String[] args) {  
          System.out.println(ConstClass.HELLOWORLD);  
      }   
}
```
`输出：hello world`
上面的示例代码运行后也没有输出“SuperClass init！”，这是因为虽然在Java源码中引用了ConstClass类中的常量HELLOWORLD，`但是在编译阶段将此常量的值“hello world”存储到了NotInitialization类的常量池中`，对于常量ConstClass.HELLOWORLD的引用实际上都被转化为NotInitialization类对自身常量池的引用了。实际上NotInitialization的Class文件之中已经不存在ConstClass类的符号引用入口了。

### 接口加载与类加载的区别
接口的加载过程与类加载的区别在于当类在初始化时要求其父类都已经初始化过了，但是一个接口在初始化时，并不要求其父类都完成了初始化，只有在真正用到父类接口的时候（如引用父接口的常量）才会初始化。

#### 示例
主类NotInitialization.java
```java
package pac1;

/**
 * Created by vino on 2016/5/19.
 */
public class NotInitialization {
  public static void main(String[] args) {  
          new SubClass();
      }   
}
```
输出结果：
```java
SuperClass init!
SubClass init!
```

如果此时的父类是一个接口，则只输出`SubClass init!`。

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 

