---
title: 根据字节码探讨JAVA自增运算符的原理
date: 2016-08-29 10:47:24
tags: 
- java
- 字节码
categories: java
---
```java
public class Test {
    
    static int x, y;

    public static void main(String args[]) {
        x++;
        myMethod();
        System.out.println(x + y + ++x);
    }

    public static void myMethod() {
        y = x++ + ++x;
    }
}
```
**如果以上代码的结果你很自信能做对,那么本文或许对你帮助不大,但仍然可以看下java底层的实现.在最后将给出以上代码的结果以及解析.**
#### 情况举例
本文中的例子主要针对以下情况:

①x=y++

②x=++y

③x=x++

④x=++x

a : x,y为形参

b : x,y为成员变量

废话不多说,直接上代码:

#### 代码1(①+b):
```java
public   class Test {

    static int x,y;
    public static void main(String args[]){
        test();
    }

    public static void test(){
        x = y++;
        System.out.println(x+""+y);
    }
}
```
`结果:01`

**test()字节码:**

```java
0: getstatic     #3                  // Field y:I
         3: dup
         4: iconst_1
         5: iadd
         6: putstatic     #3                  // Field y:I
         9: putstatic     #4                  // Field x:I
        12: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: getstatic     #4                  // Field x:I
        18: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
        21: return
```
解释:

0:y(值)入操作栈

3:得到y(值)的一个快照y'(值)

(个人认为相当于是将栈顶元素也就是y(值)复制了一份,然后将复制得到的y'(值)入操作栈,现在操作栈中有y(值)和y'(值))

4:常量1入操作栈

5:常量1和y'(值)弹出栈,进行加操作,并将结果s入栈

6:将结果s弹出栈,赋给y(变量)(此时y==1)

9:将y(值)弹出栈,赋给x(变量)(此时x==0)

因为y(值)入操作栈之后没有修改,所以x依旧是0,而y变成了1

#### 代码2(②+b):

```java
public   class Test {

    static int x,y;
    public static void main(String args[]){
        test();
    }

    public static void test(){
        x = ++y;
        System.out.println(x+""+y);
    }
}
```

`结果11`

**test()字节码:**

```java
0: getstatic     #3                  // Field y:I
         3: iconst_1
         4: iadd
         5: dup
         6: putstatic     #3                  // Field y:I
         9: putstatic     #4                  // Field x:I
        12: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: getstatic     #4                  // Field x:I
        18: invokevirtual #6                  // Method java/io/PrintStream.println:(I)V
        21: return
```
解释:

0:y(值)入操作栈

3:常量1入操作栈

4:常量1和y(值)弹出栈,进行加操作,并将结果s入栈

5:得到栈顶元素也就是s的快照s',并入操作栈

6:将s'弹出栈,并赋给y(变量)(此时y==1)

9:将s弹出栈,并赋给x(变量)(此时x==1)

#### 代码3(③+b):

```java
public   class Test {

    static int x;
    public static void main(String args[]){
        test();
    }

    public static void test(){
        x = x++;
        System.out.println(x);
    }
}
```
`结果:0`

**test()字节码:**

```java
0: getstatic     #3                  // Field x:I
         3: dup
         4: iconst_1
         5: iadd
         6: putstatic     #3                  // Field x:I
         9: putstatic     #3                  // Field x:I
        12: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: getstatic     #3                  // Field x:I
        18: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        21: return
```

解释:

0:x(值)入操作栈

3:得到x(值)的快照x',入操作栈

4:常量1入操作栈

5:常量1和x'弹出操作栈,进行加操作,将结果s入操作栈

6:将s弹出栈,并赋给x(变量)(此时x==1)

9:将x(值)弹出栈,并赋给x(变量)(此时x的值被覆盖,x==0)

#### 代码4(④+b):

```java
public   class Test {

    static int x;
    public static void main(String args[]){
        test();
    }

    public static void test(){
        x = ++x;
        System.out.println(x);
    }
}
```
`结果:1`

**test()字节码:**

```java
0: getstatic     #3                  // Field x:I
         3: iconst_1
         4: iadd
         5: dup
         6: putstatic     #3                  // Field x:I
         9: putstatic     #3                  // Field x:I
        12: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: getstatic     #3                  // Field x:I
        18: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        21: return
```
解释:

0:x(值)入操作栈

3:常量1如入操作栈

4:常量1和x(值)弹出操作栈,进行加操作,并将结果s入操作栈

5:得到栈顶元素s的快照s',入操作栈

6.将s'弹出操作栈,并赋给x(变量)(此时x==1)

9:将s弹出操作栈,并赋给x(变量)(此时x==1)

#### 代码5(①+a):

```java
public   class Test {

    public static void main(String args[]){
        test(0,0);
    }

    public static void test(int x,int y){
        x = y++;
        System.out.println(x+""+y); } }
```
`结果:01`

**test()字节码:**

```java
0: iload_1
         1: iinc          1, 1
         4: istore_0
         5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: new           #4                  // class java/lang/StringBuilder
        11: dup
        12: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
        15: iload_0
        16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        19: ldc           #7                  // String
        21: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: iload_1
        25: invokevirtual #6                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        28: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        31: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        34: return
```
解释:

0:将本地变量区的y(值)入操作栈

1:将本地变量y加1(y==1)

4:将0中的y(值)弹出栈,并赋给本地变量区的x(x==0)

#### 代码6(②+a):

```java
public   class Test {

    public static void main(String args[]){
        test(0,0);
    }

    public static void test(int x,int y){
        x = ++y;
        System.out.println(x+""+y);
    }
}
```
`结果:11`

**test()字节码:**

```java
0: iinc          1, 1
         3: iload_1
         4: istore_0
         5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: new           #4                  // class java/lang/StringBuilder
        11: dup
        12: invokespecial #5                  // Method java/lang/StringBuilder."<init>":()V
        15: iload_0
        16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        19: ldc           #7                  // String
        21: invokevirtual #8                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: iload_1
        25: invokevirtual #6                  // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
        28: invokevirtual #9                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        31: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        34: return
```
解释:

0:将本地变量y加1(此时y==1)

3:将本地变量y(值)入操作栈

4:将y(值)弹出操作栈,并赋给x(此时x==1)

#### 代码7(③+a):

```java
public   class Test {

    public static void main(String args[]){
        test(0);
    }

    public static void test(int x){
        x = x++;
        System.out.println(x);
    }
}
```
`结果:0`

**test()字节码:**

```java
0: iload_0
         1: iinc          0, 1
         4: istore_0
         5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: iload_0
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        12: return
```
解释:

0:本地变量x(值)入操作栈

1:本地变量x加1(此时x==1)

4:将x(值)弹出栈,并赋给本地变量x(此时x==0)

#### 代码8(④+a):

```java
public   class Test {

    public static void main(String args[]){
        test(0);
    }

    public static void test(int x){
        x = ++x;
        System.out.println(x);
    }
}
```
`结果:1`

**test()字节码:**

```java
0: iinc          0, 1
         3: iload_0
         4: istore_0
         5: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: iload_0
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        12: return
```
解释:

0:本地变量x加1(此时x==1)

3:本地变量x(值)入操作栈

4:将x(值)弹出操作栈,并赋给x(此时x==1) 

----
#### 小结

<font color= red>事实上i++和++i在底层的实现都是先自增,区别在于返回值.i++返回自增前的值,++i返回自增后的值。</font>

 

现在来看看一开始那段代码的结果和解析:

`结果:11`

**myMethod()字节码:**

```java
0: getstatic     #2                  // Field x:I
         3: dup
         4: iconst_1
         5: iadd
         6: putstatic     #2                  // Field x:I
         9: getstatic     #2                  // Field x:I
        12: iconst_1
        13: iadd
        14: dup
        15: putstatic     #2                  // Field x:I
        18: iadd
        19: putstatic     #5                  // Field y:I
        22: return
```
解释:

0:变量x(值)入操作栈(栈状态:x0)

3:得到栈顶元素x(值)的快照x'(值),并入操作栈(栈状态:x0->x0')

4:常量1入操作栈(栈状态:x0->x0'->1)

5:常量1和x'(值)弹出操作栈,进行加操作,将结果s0入操作栈(栈状态:x0->s0)

6:弹出s0,并赋给x(变量)(栈状态:x0,此时x(变量)==2)

9:将修改后的x(值)入操作栈(栈状态:x0->x1)

12:常量1入操作栈(栈状态:x0->x1->1)

13:常量1和X1(值)弹出操作栈,进行加操作,将结果s1入操作栈(栈状态:x0->s1)

14:得到栈顶元素s1(值)的快照s1'(值),并入操作栈(栈状态:x0->s1->s1')

15:弹出s1'并赋给x(变量)(栈状态:x0->s1)

18:s1和x0弹出栈,进行加操作,将结果s2入栈(栈状态:s2)

19:弹出s2,并赋给y(变量) 

所以在myMethod之后x的值为经过两次自增后的值,为x+2==3,而y的值为x0+s1,其中x0为最初传进来的值==1,s1是x经过两次自增后的值==3,所以y==4

x==3,y==4

最后输出的结果就是3+4+4==11

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 