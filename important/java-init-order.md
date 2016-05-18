---
title: java类静态域、块，非静态域、块，构造函数的初始化顺序
date: 2016-05-18 10:21:11
tags: java
categories: java
photos:
- /images/java/java-common-head.jpg
---
#### 开始
面试的时候，经常会遇到这样的考题：给你两个类的代码，它们之间是继承的关系，每个类里只有构造器方法和一些变量，

构造器里可能还有一段代码对变量值进行了某种运算，另外还有一些将变量值输出到控制台的代码，然后让我们判断输出的
结果。这实际上是在考查我们对于继承情况下类的初始化顺序的了解。
我们大家都知道，对于静态变量、静态初始化块、变量、初始化块、构造器，它们的初始化顺序以此是
（静态变量、静态初始化块）>（变量、初始化块）>构造器。
我们也可以通过下面的测试代码来验证这一点：  
Java代码:
```java
public class InitialOrderTest {               
    // 静态变量          
    public static String staticField = "静态变量";          
    // 变量          
    public String field = "变量";                  
    // 静态初始化块        
    static {          
        System.out.println(staticField);  
        System.out.println("静态初始化块");  
    }                  
    // 初始化块              
    {          
        System.out.println(field);  
        System.out.println("初始化块");        
    }                  
    // 构造器          
    public InitialOrderTest() {  
        System.out.println("构造器");     
    }                  
    public static void main(String[] args) {  
        new InitialOrderTest();              
    }          
}
```
运行以上代码，我们会得到如下的输出结果：
```
静态变量  
静态初始化块  
变量  
初始化块  
构造器
```
这与上文中说的完全符合。
那么对于继承情况下又会怎样呢？我们仍然以一段测试代码来获取最终结果：
Java代码 :
```java
class Parent {              
    // 静态变量          
    public static String p_StaticField = "父类--静态变量";              
    // 变量          
    public String p_Field = "父类--变量";                  
    // 静态初始化块              
    static {          
        System.out.println(p_StaticField);                  
        System.out.println("父类--静态初始化块");              
    }                  
    // 初始化块              
    {          
        System.out.println(p_Field);          
        System.out.println("父类--初始化块");              
    }                  
    // 构造器          
    public Parent() {          
        System.out.println("父类--构造器");              
    }         
}                  
public class SubClass extends Parent {              
    // 静态变量          
    public static String s_StaticField = "子类--静态变量";              
    // 变量          
    public String s_Field = "子类--变量";              
    // 静态初始化块              
    static {          
        System.out.println(s_StaticField);                  
        System.out.println("子类--静态初始化块");          
    }          
    // 初始化块   
    {          
        System.out.println(s_Field);          
        System.out.println("子类--初始化块");              
    }                  
    // 构造器          
    public SubClass() {          
        System.out.println("子类--构造器");              
    }                  
    // 程序入口          
    public static void main(String[] args) {                  
        new SubClass();             
    }          
}
```
运行一下上面的代码，结果马上呈现在我们的眼前：

```
父类--静态变量  
父类--静态初始化块  
子类--静态变量  
子类--静态初始化块  
父类--变量  
父类--初始化块  
父类--构造器  
子类--变量  
子类--初始化块  
子类--构造器
```
现在，结果已经不言自明了。大家可能会注意到一点，那就是，并不是父类完全初始化完毕后才进行子类的初始化，
实际上子类的静态变量和静态初始化块的初始化是在父类的变量、初始化块和构造器初始化之前就完成了。
那么对于静态变量和静态初始化块之间、变量和初始化块之间的先后顺序又是怎样呢？
是否静态变量总是先于静态初始化块，变量总是先于初始化块就被初始化了呢？实际上这取决于它们在类中出现的先后顺序。
我们以静态变量和静态初始化块为例来进行说明。 同样，我们还是写一个类来进行测试：    

Java代码:
```java
public class TestOrder {          
    // 静态变量          
    public static TestA a = new TestA();                       
    // 静态初始化块              
    static {          
        System.out.println("静态初始化块");              
    }                       
    // 静态变量          
    public static TestB b = new TestB();                  
    public static void main(String[] args) {                  
        new TestOrder();              
    }          
}                  
class TestA {              
    public TestA() {          
        System.out.println("Test--A");              
    }          
}                  
class TestB {              
    public TestB() {          
        System.out.println("Test--B");              
    }          
}
```
运行上面的代码，会得到如下的结果：
```
Test--A  
静态初始化块  
Test--B
```
大家可以随意改变变量a、变量b以及静态初始化块的前后位置，就会发现输出结果随着它们在类中出现的前后顺序而改变，
这就说明静态变量和静态初始化块是依照他们在类中的定义顺序进行初始化的。同样，变量和初始化块也遵循这个规律。
了解了继承情况下类的初始化顺序之后，如何判断最终输出结果就迎刃而解了。  

测试函数：  
```java
public class TestStaticCon {   
    public static int a = 0;  
      static {    
        a = 10;  
          System.out.println("父类的静态代码块在执行a=" + a);   
    }     
    {    
        a = 8;  
          System.out.println("父类的非静态代码块在执行a=" + a);   
    }  
      public TestStaticCon() {   

        this("a在父类带参构造方法中的值：" + TestStaticCon.a); // 调用另外一个构造方法    
        System.out.println(a);    
        System.out.println("父类无参构造方法在执行a=" + a);   
    }  
      public TestStaticCon(String n) {    
        System.out.println(n);    
        System.out.println(a);  
      }  
      public static void main(String[] args) {    
        TestStaticCon tsc = null;  
          System.out.println("!!!!!!!!!!!!!!!!!!!!!");    
        tsc = new TestStaticCon();   
    }  
}
```
运行结果：
```
父类的非静态代码块在执行a=10
!!!!!!!!!!!!!!!!!!!!!
父类的非静态代码块在执行a=8
a在父类带参构造方法中的值：10
8
8
父类无参构造方法在执行a=8  
```
#### 结论
静态代码块是在类加载时自动执行的，非静态代码块是在创建对象时自动执行的代码，不创建对象不执行该类的非静态代码块。且执行顺序为静态代码块------非静态代码块----构造函数。
扩展：静态代码块  与  静态方法：
一般情况下,如果有些代码必须在项目启动的时候就执行的时候,需要使用静态代码块,这种代码是主动执行的;
需要在项目启动的时候就初始化,在不创建对象的情况下,其他程序来调用的时候,需要使用静态方法,这种代码是被动执行的.  
两者的区别就是:静态代码块是自动执行的;  静态方法是被调用的时候才执行的.  

作用:静态代码块可用来初始化一些项目最常用的变量或对象;静态方法可用作不创建对象也可能需要执行的代码

#### 阿里笔试题

求下面这段代码的输出：
```java
public class Test1 {  

    public static int k = 0;  
    public static Test1 t1 = new Test1("t1");  
    public static Test1 t2 = new Test1("t2");  
    public static int i = print("i");  
    public static int n = 99;  
    public int j = print("j");  
    {  
        print("构造块");  
    }  

    static{  
        print("静态块");  
    }  

    public Test1(String str){  
        System.out.println((++k)+":"+str+"    i="+i+"    n="+n);  
        ++i;++n;  
    }  

    public static int print(String str){  
        System.out.println((++k)+":"+str+"    i="+i+"    n="+n);  
        ++n;  
        return ++i;  
    }  

    public static void main(String[] args) {  
        // TODO Auto-generated method stub  
        Test1 t = new Test1("init");  
    }  

}
```
运行结果：
```
j    i=0    n=0
构造块    i=1    n=1
t1    i=2    n=2
j    i=3    n=3
构造块    i=4    n=4
t2    i=5    n=5
i    i=6    n=6
静态块    i=7    n=99
j    i=8    n=100
构造块    i=9    n=101
init    i=10    n=102
```

原文：http://ini.iteye.com/blog/2007835
