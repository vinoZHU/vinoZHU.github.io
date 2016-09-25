---
title: JNI探秘--FileDescriptor、FileInputStream 解惑
date: 2016-09-14 09:13:07
tags: 
- java
- JNI
categories: 
- java
- JNI
---
#### 介绍

使用JAVA读取文件时需要用到`FileInputStream`这个类，最简单的使用方式如下:

```java
public static void main(String[] args){
        try {
            FileInputStream fileInputStream = new FileInputStream("test.txt");
            System.out.println(fileInputStream.read());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
```
`FileInputStream`源码中的构造方法一共有3个：

```java
public FileInputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null);
    }

public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);
        path = name;
        open(name);
    }
    

public FileInputStream(FileDescriptor fdObj) {
        SecurityManager security = System.getSecurityManager();
        if (fdObj == null) {
            throw new NullPointerException();
        }
        if (security != null) {
            security.checkRead(fdObj);
        }
        fd = fdObj;
        path = null;

        /*
         * FileDescriptor is being shared by streams.
         * Register this stream with FileDescriptor tracker.
         */
        fd.attach(this);
    }    
```

`fd`和`path`定义如下:

```java
	/* File Descriptor - handle to the open file */
	private final FileDescriptor fd;

   /**
    * The path of the referenced file
    * (null if the stream is created with a file descriptor)
    */
   private final String path;
```

参数为`String name`或者`File file`的构造方法都新建了一个`fileDescriptor`,并赋值给`fd`，而参数为`FileDescriptor fdObj`的构造方法直接将`fdObj`参数赋值给`fd`。其实从这里可以感觉出`FileDescriptor`(文件描述符)是JAVA中的文件操作核心。

#### 疑惑一

```java
public static void main(String[] args) throws IOException, NoSuchFieldException {

        FileDescriptor fileDescriptor = null;
        FileDescriptor fileDescriptor1 = null;

        FileInputStream fileInputStream = new FileInputStream("test.txt");
        FileInputStream fileInputStream1 = new FileInputStream("test.txt");
        
        System.out.println(fileInputStream.getFD().valid());
        System.out.println(fileInputStream1.getFD().valid());
        fileDescriptor = fileInputStream.getFD();
        fileDescriptor1 = fileInputStream1.getFD();
        
    }
```
输出:

```
true
true
```
其中`fileDescriptor `和`fileDescriptor1`的值分别为:
![](/images/JNI/FileDescriptor-and-FileInputStream-0.png)
查看`FileDescriptor`的源码，2个构造方法如下:

```java
    public  FileDescriptor() {
        fd = -1;
    }

    private  FileDescriptor(int fd) {
        this.fd = fd;
    }
```
在`FileDescriptor`中的`fd`是一个`int`类型的值。`FileDescriptor`源码中只有一个public的构造方法，而且`fd`的初始值为`-1`,但是`FileInputStream`中的`fd`(FileDescriptor类型)的`fd`值通过调试看到不为`-1`（2个fd是包含的关系）。输出`true`的条件就是`fd != -1`。

#### 疑惑一解密

在参数为`File file`的构造方法的最后调用了一个`open`方法，初步怀疑在这个方法内改变了`fd`的内容。通过打断点调试，的确在`open`方法调用之后`fd`的值改变了。`open`方法的最终调用为：

```java
    private native void open0(String name) throws FileNotFoundException;

```
可以看到是一个`native`方法，对应的JNI方法如下:

```java
JNIEXPORT void JNICALL
Java_java_io_FileInputStream_open(JNIEnv *env, jobject this, jstring path) {
    fileOpen(env, this, path, fis_fd, O_RDONLY);
}
```
`env`是JNI的一个对象，`this`表示调用`open`方法的`FileInputStream`对象，`path`为传进来的参数（文件名）,`O_RDONLY`表示只读，`fis_fd`是在JNI中定义的一个变量:

```c
jfieldID fis_fd; /* id for jobject 'fd' in java.io.FileInputStream */

/**************************************************************
 * static methods to store field ID's in initializers
 */

JNIEXPORT void JNICALL
Java_java_io_FileInputStream_initIDs(JNIEnv *env, jclass fdClass) {
    fis_fd = (*env)->GetFieldID(env, fdClass, "fd", "Ljava/io/FileDescriptor;");
}```
`fis_fd`通过`Java_java_io_FileInputStream_initIDs`方法初始化，该方法对应了`FileInputStream`如下代码:

```java
static {
        initIDs();
    }
```
所以，在`FileInputStream`类加载阶段，`fis_fd`就被初始化了，`fid_fd`相当于是`fd`字段的一个内存偏移量。`open`方法直接调用了`fileOpen`方法，`fileOpen`方法如下:

```c
void
fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
    WITH_PLATFORM_STRING(env, path, ps) {
        FD fd;

#if defined(__linux__) || defined(_ALLBSD_SOURCE)
        /* Remove trailing slashes, since the kernel won't */
        char *p = (char *)ps + strlen(ps) - 1;
        while ((p > ps) && (*p == '/'))
            *p-- = '\0';
#endif
        fd = handleOpen(ps, flags, 0666);
        if (fd != -1) {
            SET_FD(this, fd, fid);
        } else {
            throwFileNotFoundException(env, path);
        }
    } END_PLATFORM_STRING(env, ps);
}
```
其中的`handleOpen`函数打开了一个文件句柄（一个数字），相当于和文件建立了联系，并且将返回的句柄赋值给了局部变量`fd`,然后调用了`SET_FD`宏:

```c
#define SET_FD(this, fd, fid) \
    if ((*env)->GetObjectField(env, (this), (fid)) != NULL) \
        (*env)->SetIntField(env, (*env)->GetObjectField(env, (this), (fid)),IO_fd_fdID, (fd))
```
该函数首先判断FileInputStream这个对象的fd属性是不是空，如果不为空，则进行赋值。`fd`是刚得到的文件句柄，`(*env)->GetObjectField(env, (this), (fid))`是`FileInputStream`对象的`fd`字段。但是句柄`fd`是`int`类型的，而`FileInputStream`对象的`fd`字段是`FileDescriptor`类型的，如何赋值?理所当然，我们需要一个偏移量，一个`FileDescriptor`中的`fd`字段的偏移量，也就是`IO_fd_fdID`的值。`IO_fd_fdID`是在`FileDescriptor`对应JNI代码的一个变量，在类加载时期初始化,通过静态代码块:

```java
static {
        initIDs();
    }
```
对应的`native`方法如下:

```c
/* field id for jint 'fd' in java.io.FileDescriptor */
jfieldID IO_fd_fdID;

/**************************************************************
 * static methods to store field ID's in initializers
 */

JNIEXPORT void JNICALL
Java_java_io_FileDescriptor_initIDs(JNIEnv *env, jclass fdClass) {
    IO_fd_fdID = (*env)->GetFieldID(env, fdClass, "fd", "I");
}
```

由此可得，调用`open`方法之后，`FileInputSream`对象的`fd`的值被改变了。

#### 疑惑二
既然`FileDescriptor`是文件操作的核心，那么`read`方法调用又是怎么和它联系起来的？

#### 疑惑二解密

`FileInputStream`中的`read`方法:

```java
public int read() throws IOException {
        return read0();
    }

    private native int read0() throws IOException;
```
对应的`native`方法:

```c
JNIEXPORT jint JNICALL
Java_java_io_FileInputStream_read(JNIEnv *env, jobject this) {
    return readSingle(env, this, fis_fd);
}
```
`readSingle()`方法:

```c
jint
readSingle(JNIEnv *env, jobject this, jfieldID fid) {
    jint nread;
    char ret;
    FD fd = GET_FD(this, fid);
    if (fd == -1) {
        JNU_ThrowIOException(env, "Stream Closed");
        return -1;
    }
    nread = IO_Read(fd, &ret, 1);
    if (nread == 0) { /* EOF */
        return -1;
    } else if (nread == -1) { /* error */
        JNU_ThrowIOExceptionWithLastError(env, "Read error");
    }
    return ret & 0xFF;
}
```

虽然java代码中没有表现出对`fd`的使用，但是在`native`代码中的确使用了`fd`。

#### 总结
JAVA中的文件操作最终都是要通过`FileDescriptor`，在Unix/Linux中的文件描述符就是一个数字，对应了进程打开文件数组的下标，该数组的0,1,2号文件分别表示标准输入、标准输出，标准错误输出。这和JAVA中是一致的，`FileDescriptor`中的fd为0，1，2时也表示同样的意义。所以以下代码也可以用于输出`'A'`:

```java
FileOutputStream fileOutputStream = new FileOutputStream(FileDescriptor.out);
        fileOutputStream.write('A');
```

当我们通过`文件名`或者`文件对象`new一个FileInputStream的时候，做了以下步骤：

1. 如果FileInputStream类尚未加载，则执行initIDs方法，否则这一步直接跳过。

2. 如果FileDescriptor类尚未加载，则执行initIDs方法，否则这一步也直接跳过。

3. new一个FileDescriptor对象赋给FileInputStream的fd属性。

4. 打开一个文件句柄。

5. 将文件句柄赋给FileDescriptor对象的fd字段。

注：本文JDK版本为1.8

><font color= Darkorange>如若觉得本文尚可，欢迎转载交流。转载请在正文明显处注明[原站地址](http://vinoit.me)以及原文地址，谢谢！</font> 